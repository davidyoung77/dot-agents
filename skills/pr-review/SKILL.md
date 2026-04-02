---
name: pr-review
description: |
  Review a GitHub pull request with full local validation and multi-model consensus.
  Use when user shares a PR URL, says "review this PR", or asks to review a pull request.
  Handles enterprise SSO repos (uses gh CLI, not FetchUrl), reviews in an isolated
  git worktree, reconciles prior feedback, runs validation/tests, dispatches code
  reviewers on every PR plus spec/PRD reviewers when warranted, consolidates findings,
  and posts inline review comments on the correct diff lines.
---

# PR Review

End-to-end pull request review: resolve PR context, review in an isolated worktree,
validate locally, collect multi-model feedback, present findings, then optionally
post inline comments where they belong.

## Mode Detection

| User says | Action |
|---|---|
| "review this PR", "review PR #N", shares a PR URL | Full review flow |
| "just run the tests on this PR" | Skip droid dispatch and posting; run local validation only |
| "post the review" | Post previously approved findings if a saved report/payload exists |

## Flow

### 0. Resolve Context and Review Artifacts

Use `gh` CLI (NOT FetchUrl -- enterprise repos hit SSO walls):

```bash
gh pr view <number> --json title,body,files,commits,headRefName,baseRefName,state,additions,deletions
gh pr diff <number>
```

1. Determine the repository and PR number:
   - If the user shared a PR URL, use it directly or extract the PR number from it.
   - Otherwise try `gh pr view --json number --jq '.number'` from the current branch.
   - If neither works, ask the user for the PR number.
2. Determine the GitHub `owner/repo` slug from `git remote get-url origin`.
3. Determine `<REPO_NAME>` from `git rev-parse --show-toplevel` for local artifact paths.
4. Resolve artifact locations:
   - Worktree path: `~/git/worktrees/<REPO_NAME>/pr-<NUMBER>`
   - Report directory: first look for agent output/work artifact guidance in `AGENTS.md`, `.agents/AGENTS.md`, `.factory/AGENTS.md`, or `.agent/AGENTS.md`
   - If no repo-specific artifact directory is defined, fall back to `.factory/pr-reviews/reports/`
5. If using the fallback `.factory/pr-reviews/` path, ensure it is gitignored before writing artifacts there.
6. Fetch and retain:
   - PR metadata
   - full diff
   - changed file list
   - diff stats
   - linked tickets if present
7. Print a short summary for the user:
   - PR title and number
   - base <- head
   - additions/deletions and changed-file count
   - PR URL

### 1. Assess Prior Review Feedback

Before starting a new review pass, inspect existing review feedback on the PR.

1. Fetch:
   - Inline review comments: `gh api repos/{owner}/{repo}/pulls/{number}/comments --paginate`
   - Top-level reviews: `gh api repos/{owner}/{repo}/pulls/{number}/reviews --paginate`
2. If both are empty, skip this step.
3. Classify each prior comment thread as best you can:
   - resolved by code change
   - resolved by author reply
   - outdated
   - unresolved
4. Present a compact summary before continuing.
5. If unresolved feedback exists, pass it into the new review scope under a `Prior Unresolved Feedback` section.
6. If status is unclear, treat the item as unresolved rather than assuming it is handled.

### 2. Set Up Isolated Worktree

Review in a detached worktree. Do not `git checkout` the PR branch in the user's current working tree.

```bash
git fetch origin <headRefName>
git worktree add ~/git/worktrees/<REPO_NAME>/pr-<NUMBER> origin/<headRefName> --detach
```

1. If the target worktree path already exists, remove and recreate it only if it is clearly disposable.
2. Run dependency installation inside the worktree using the project-appropriate package manager.
3. If worktree creation fails, warn the user and ask before falling back to a diff-only review.
4. If the PR diff exceeds roughly 500 lines, save it to `<WORKTREE_PATH>/pr-<NUMBER>-diff.txt` and use that file path in reviewer prompts instead of inlining the full diff.
5. Keep the user's main checkout untouched throughout the review.

### 3. Static Validation

Detect and run the project's validation commands inside the worktree. Check for:
- `package.json` scripts: `typecheck`, `lint`, `build`, `bddgen`
- Common patterns: `tsc --noEmit`, `eslint`, `prettier --check`

Run ALL that exist. Report results.

### 4. Test Execution

Run the project's test suite inside the worktree. Capture:
- Total tests, passed, failed, skipped
- Failure messages with file/line references
- Whether failures appear introduced by the PR or are likely pre-existing

If tests require auth/credentials and fail at auth step:
- Check for project-specific refresh commands (`.factory/commands/refresh-tokens.md`)
- Check for auth helper scripts
- Report the auth blocker clearly -- do NOT silently skip tests

Only compare against the base branch when needed to disambiguate whether a failure is pre-existing.

### 5. Multi-Model Review (Code Always, Spec/PRD Conditionally)

Select the reviewer matrix before dispatching:

**Code reviewers** run on every PR:
- `code-reviewer-opus`
- `code-reviewer-gpt`
- `code-reviewer-gemini`

**Spec reviewers** run only when architectural/spec drift review is warranted, for example:
- blueprints/specs exist for the touched area
- the PR changes architecture, service boundaries, APIs, schemas, migrations, or data flow
- the PR is cross-cutting or spans multiple modules/packages
- the user explicitly asks for deeper architecture/spec review

When spec review is warranted, run:
- `spec-reviewer-opus`
- `spec-reviewer-gpt`
- `spec-reviewer-gemini`

**PRD reviewers** run only when requirements/behavior coverage review is warranted, for example:
- requirements docs/PRDs exist for the touched feature area
- the PR changes user-visible behavior, workflows, permissions, or business rules
- the PR is intended to satisfy acceptance criteria that should be checked back against requirements
- the user explicitly asks for requirements coverage review

When PRD review is warranted, run:
- `prd-reviewer-opus`
- `prd-reviewer-gpt`
- `prd-reviewer-gemini`

If optional reviewer families are skipped, note that explicitly in the final summary and report.

Each gets the same core context:
- PR number, title, URL, base branch, and head branch
- PR description/body
- absolute worktree path
- full changed-file list
- static validation summary
- test summary
- prior unresolved feedback, if any
- the diff inline for small PRs, or the saved diff file path for large PRs

### Reviewer Prompt Contract

When constructing the prompt for each reviewer, instruct them to:
- Read the full changed files in the worktree for context, not just the diff hunks
- Use the diff to identify what changed, but keep actionable findings focused on added or modified code
- Mention untouched-code concerns only as observations or notes unless they are directly required to explain a defect in changed code
- Verify file and line references against the HEAD version of the changed files or the diff; never invent line numbers
- Avoid reporting local environment, auth, or test-lab failures as code findings
- Include exact replacement-style fix suggestions only when the fix is small, local, and copy-pasteable
- Confirm prior unresolved comments as resolved notes when the current code actually addresses them, rather than reflexively reopening them
- Return structured findings with severity, file, line, explanation, and optional suggested fix
- Never make edits

Handle timeouts gracefully -- if a droid times out, note it and continue with results from those that completed.

### 6. Consolidate Findings

Normalize reviewer-specific outputs into a common findings list, then merge:
- Code reviewer `must_fix` findings map to the highest severities
- Code reviewer `should_fix` findings map to medium/high severities
- Spec reviewer gaps and risks become findings with clear severity and file/context references where available
- PRD reviewer gaps, contradictions, and requirements-coverage risks become findings with clear severity and document context where available
- Suggestions and positive observations remain non-blocking items

Then consolidate:
- **Consensus items**: flagged by 2+ droids -> higher confidence
- **Unique items**: flagged by only 1 droid -> still include but note
- Deduplicate (same finding on same line from multiple droids)
- When multiple droids assess the same location differently, keep the majority assessment and note dissent
- Group by severity: Critical > High > Medium > Low
- Include positive findings too (good patterns, nice changes)

### 7. Write a Durable Review Report

Before asking to post, write a markdown review report to `<REPORT_DIR>/pr-<NUMBER>-review.md`.

Include:
- PR metadata and URL
- validation and test summaries
- prior-feedback summary
- reviewer matrix used, including any optional reviewer families that were skipped
- consolidated findings grouped by severity
- which droids agreed on each finding
- the proposed review action: `APPROVE`, `REQUEST_CHANGES`, or `COMMENT`

Save large intermediate artifacts when useful:
- raw reviewer outputs
- saved diff file for large PRs
- review payload JSON before posting

### 8. Present to User for Approval

Show the consolidated review BEFORE posting. Include:
- Test results summary table
- Prior-feedback summary if relevant
- Reviewer matrix used, including any conditional reviewer families that were skipped
- Findings grouped by severity with file:line references
- Which droids agreed on each finding
- Proposed review action (APPROVE, REQUEST_CHANGES, COMMENT)
- Review report path

Ask: "Ready to post this review, or want to adjust anything?"

### 9. Post Inline Review

Use the GitHub API to post a proper review with inline comments:

```bash
gh api repos/{owner}/{repo}/pulls/{number}/reviews --method POST --input review.json
```

The review JSON structure:
```json
{
  "commit_id": "<head SHA>",
  "event": "REQUEST_CHANGES|APPROVE|COMMENT",
  "body": "Summary with test results table",
  "comments": [
    {
      "path": "relative/file/path.ts",
      "line": <line number in NEW file>,
      "body": "**Severity: Finding title**\n\nDetails and fix suggestion."
    }
  ]
}
```

**Critical: Line numbers must be within the diff.** For new files (status: "added"), any line works. For modified files, only lines in diff hunks are valid. Check with:
```bash
gh api repos/{owner}/{repo}/pulls/{number}/files --jq '.[] | {filename, patch}'
```

If a finding is on a line outside the diff, use `subject_type: "file"` to comment on the file generally.
If a finding spans multiple lines and has an exact replacement fix, use `start_line` plus `line` so GitHub can render a multi-line suggestion block correctly.

## Comment Format

Each inline comment should follow this pattern:
```
**{Severity}: {Title}**

{Description of the issue}

**Fix:** {Actionable suggestion}
```

Severity levels:
- **Critical** -- must fix before merge (broken functionality, security issues)
- **High** -- strongly recommended (will cause problems in production/CI)
- **Medium** -- should fix (anti-patterns, code quality, maintainability)
- **Low** -- nice to have (style, minor improvements)
- Positive comments use :+1: prefix

### 10. Cleanup and Recovery

1. Remove the detached worktree when review work is complete.
2. Keep the markdown report and any saved payloads/artifacts in the report directory.
3. If posting fails, keep the review payload on disk and show the retry command.

## Important Notes

- **Never post without user approval.** Always present findings first.
- **Prefer worktrees over checkout/stash.** Do not disturb the user's current branch/state unless they explicitly ask.
- **Don't post a wall-of-text comment.** Use inline comments on specific lines where possible.
- **Include test results** in the review body summary.
- **Note security findings prominently** -- hardcoded secrets, exposed tokens, etc.
- **Positive feedback matters** -- call out good patterns, not just problems.
- **Handle auth gracefully** -- if tests can't run due to expired tokens, say so clearly and check for project-specific refresh mechanisms.
- **Use saved reports for follow-up.** If the user later says "post the review", reuse the latest approved findings/report when possible instead of recomputing everything.

## Error Handling

- If `gh` is not authenticated, instruct the user to run `gh auth login`.
- If a reviewer droid fails or times out, continue with the remaining results and note the missing reviewer.
- If the worktree cannot be created, ask whether to continue with a diff-only review.
- If validation or tests fail due to environment/auth setup, report the blocker clearly and do not silently skip the step.
