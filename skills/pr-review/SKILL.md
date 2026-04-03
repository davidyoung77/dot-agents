---
name: pr-review
description: |
  Review a GitHub pull request with local validation, targeted runtime validation
  when warranted, PDLC-aligned audit trail capture, and capability-aware reviewer
  consensus.
  Use when user shares a PR URL, says "review this PR", or asks to review a pull request.
  Handles enterprise SSO repos (uses gh CLI, not FetchUrl), reviews in an isolated
  git worktree, reconciles prior feedback, runs validation/tests and real-surface checks
  when warranted, dispatches code
  reviewers on every PR plus spec/PRD reviewers when warranted, consolidates findings,
  and posts inline review comments on the correct diff lines.
---

# PR Review

End-to-end pull request review: resolve PR context, review in an isolated worktree,
validate locally, run targeted runtime validation when warranted, capture audit-trail
and drift context, collect reviewer feedback, present findings, then optionally
post inline comments where they belong.

## Mode Detection

| User says | Action |
|---|---|
| "review this PR", "review PR #N", shares a PR URL | Full review flow |
| "just run the tests on this PR" | Skip droid dispatch and posting; run local validation only unless the user explicitly asks for runtime validation |
| "post the review" | Post previously approved findings from a temporary saved report/payload when available |

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
   - If the repo defines only a base artifact/work directory, nest PR review artifacts under `pr-reviews/pr-<NUMBER>/`
   - If no repo-specific artifact directory is defined, fall back to `<REPO_ROOT>/.agents/work/pr-reviews/pr-<NUMBER>/`
   - Only if there is no natural project-local home, fall back to `~/.agents/work/pr-reviews/<owner>/<repo>/pr-<NUMBER>/`
5. Treat everything in `<REPORT_DIR>` as temporary recovery artifacts, not durable records.
   - The PR itself is the source of truth once the review is posted successfully
   - These files exist only to support approval, retry, or crash recovery
6. If using a `.agents/work/` path, ensure the whole `.agents/work/` tree is ignored before writing there.
   - Prefer `.git/info/exclude` for local-only artifacts
   - Only update tracked `.gitignore` if the repo explicitly wants a shared ignore rule
7. If the user invoked "post the review" directly, first look in `<REPORT_DIR>` for the latest saved review report and saved payload for this PR.
   - If found, treat them as the source of truth, confirm with the user, and jump to Step 9
   - If not found, tell the user no saved review exists and ask whether to run the full review flow
8. Fetch and retain:
   - PR metadata
   - full diff
   - changed file list
   - diff stats
   - linked tickets if present
9. Print a short summary for the user:
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
- Check for project-specific refresh commands (`.agents/commands/refresh-tokens.md` or legacy `.factory/commands/refresh-tokens.md`)
- Check for auth helper scripts
- Report the auth blocker clearly -- do NOT silently skip tests

Only compare against the base branch when needed to disambiguate whether a failure is pre-existing.

### 4b. Runtime / Real-Surface Validation (When Warranted)

Decide whether independent runtime validation is warranted. It usually is when the PR changes:
- user-visible UI flows
- API endpoints, webhooks, auth, session, or permissions
- integration boundaries, runtime configuration, migrations, or other runtime-sensitive behavior
- critical paths the user explicitly wants smoke-tested

When warranted:
- Prefer the least-cost real validation path available:
  - existing smoke or e2e command if the repo already provides one
  - browser-based validation for changed UI flows
  - real API requests for changed endpoints or auth flows
  - local dev / Docker Compose / documented non-prod service checks for integration-sensitive behavior
- Read `AGENTS.md`, `.agents/AGENTS.md`, README, and project commands to find the correct runtime path before inventing one
- Before launching services, using browser automation, or making credentialed requests, present a short proposed runtime validation plan and ask for approval unless the user already explicitly asked for it
- Capture a runtime validation summary:
  - what was tested
  - what passed, failed, or was skipped
  - screenshots or evidence paths if produced
  - API endpoints exercised and status codes
  - blockers and follow-up manual verification needed

If runtime validation was warranted but blocked, include that gap clearly in the final summary and in reviewer prompts.

### 4c. Audit Trail, Traceability, and Drift

Build a concise review audit trail aligned with PDLC:
- map whatever traceability exists back to:
  - `requirement/work item/blueprint/spec -> commit -> PR -> validation result`
- use the PR body, linked tickets, branch naming, commits, and changed docs to recover the chain when possible
- note when the traceability chain is weak or missing rather than silently accepting it
- record drift conclusions:
  - blueprint drift from spec reviewers
  - requirements drift from PRD reviewers
  - whether any drift appears intentional and documented vs accidental or unexplained
- record review governance details:
  - reviewer matrix used
  - prior unresolved feedback carried forward
  - what was locally validated, runtime-tested, skipped, or blocked
- capture model, token, and cost usage for reviewer runs and other agent-assisted analysis when the harness exposes it; if unavailable, mark it `not available` rather than guessing

### 5. Reviewer Fan-Out (Code Always, Spec/PRD Conditionally)

Select the reviewer matrix before dispatching:

Capability-aware routing:
- If the platform supports distinct model assignment per reviewer, run the full reviewer family and treat agreement as true cross-model signal
- If the platform does not support true per-reviewer model selection, the same reviewer family can still be useful as prompt-diverse or lens-diverse passes, but call that limitation out explicitly in the final summary and audit trail
- If scope, latency, or cost makes full fan-out low-value, reduce the number of passes and choose the most useful review lenses for the PR

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
- runtime validation summary, if any
- audit-trail summary, including traceability refs found and current drift status
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
- Return a reviewer confidence percentage: integer `0-100`
- Calibrate the percentage roughly as:
  - `90-100` -> very strong evidence and coverage, low uncertainty
  - `70-89` -> good signal, but with some gaps or assumptions
  - `40-69` -> partial coverage or material blockers; findings may still be useful
  - `0-39` -> weak signal; review is tentative and incomplete
- Include a one- or two-sentence confidence rationale tied to the actual evidence reviewed, blockers encountered, and coverage achieved
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

Also summarize reviewer-level confidence:
- Record each reviewer's confidence percentage and rationale
- Highlight confidence disagreements across reviewers
- If helpful, summarize the spread (for example min/max or rough clustering), but keep the raw per-reviewer percentages visible
- If useful, add a short synthesized note about overall review coverage, but do not replace the per-reviewer confidence matrix with a single scalar score

Also summarize the review audit trail:
- traceability refs found, missing, or ambiguous
- blueprint and requirements drift conclusions
- validation/runtime coverage, skips, and blockers
- whether the review used true cross-model diversity or only prompt-diverse passes
- token/cost usage by reviewer or phase when available
- a short note about any remaining evidence gaps the human reviewer should know

### 7. Write Temporary Recovery Artifacts

Before asking to post, write a markdown review report to `<REPORT_DIR>/pr-<NUMBER>-review.md`.

Include:
- PR metadata and URL
- audit-trail summary:
  - traceability refs
  - drift conclusions
  - token/cost usage when available
- validation, test, and runtime validation summaries
- prior-feedback summary
- reviewer matrix used, including any optional reviewer families that were skipped
- per-reviewer confidence percentages and rationales
- consolidated findings grouped by severity
- which droids agreed on each finding
- the proposed review action: `APPROVE`, `REQUEST_CHANGES`, or `COMMENT`

Save intermediate artifacts when useful:
- runtime validation evidence paths or notes, if produced
- raw reviewer outputs
- saved diff file for large PRs

If the user later says "post the review", reuse the temporary saved review report and payload when available instead of recomputing everything.

### 8. Present to User for Approval

Show the consolidated review BEFORE posting. Include:
- Audit-trail summary with traceability refs, drift status, and token/cost usage when available
- Test results summary table
- Runtime validation summary when it was run or when a warranted check was blocked
- Prior-feedback summary if relevant
- Reviewer matrix used, including any conditional reviewer families that were skipped
- Per-reviewer confidence percentages and rationales
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

Before posting, write the final review payload to `<REPORT_DIR>/pr-<NUMBER>-review-payload.json` and use that file as the retryable source of truth.

The review JSON structure:
```json
{
  "commit_id": "<head SHA>",
  "event": "REQUEST_CHANGES|APPROVE|COMMENT",
  "body": "Summary with audit trail, validation/test/runtime/drift results, reviewer confidence summary, and token usage when available",
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

1. Remove the detached worktree only after:
   - the review was posted successfully, or
   - the user explicitly decides to stop without further retries
2. If the review posts successfully, delete the temporary report, payload, raw reviewer outputs, and saved diff from `<REPORT_DIR>` by default.
   - Remove the now-empty PR-specific report directory too, when practical
   - Keep artifacts only if the user explicitly asks to retain them or if something about the post result is incomplete
3. If posting fails, approval is deferred, or some comments could not be posted, keep the saved payload/artifacts on disk, keep the worktree available for retry/debugging, and show the retry command.

## Important Notes

- **Never post without user approval.** Always present findings first.
- **Prefer worktrees over checkout/stash.** Do not disturb the user's current branch/state unless they explicitly ask.
- **Don't post a wall-of-text comment.** Use inline comments on specific lines where possible.
- **Keep the review auditable.** The posted review should preserve the key traceability, drift, and usage/cost context because local artifacts are temporary.
- **Be explicit about review diversity.** Distinguish true cross-model agreement from prompt-diverse agreement when the platform cannot assign different models per reviewer.
- **Include test results** in the review body summary.
- **Include reviewer confidence** in the review body summary.
- **Note security findings prominently** -- hardcoded secrets, exposed tokens, etc.
- **Positive feedback matters** -- call out good patterns, not just problems.
- **Handle auth gracefully** -- if tests can't run due to expired tokens, say so clearly and check for project-specific refresh mechanisms.
- **Use saved reports for follow-up.** If the user later says "post the review", reuse the latest approved findings/report when possible instead of recomputing everything.
- **Delete temp artifacts after success.** Local PR review files are for recovery, not archival, once the review is safely in the PR.

## Error Handling

- If `gh` is not authenticated, instruct the user to run `gh auth login`.
- If a reviewer droid fails or times out, continue with the remaining results and note the missing reviewer.
- If the worktree cannot be created, ask whether to continue with a diff-only review.
- If validation or tests fail due to environment/auth setup, report the blocker clearly and do not silently skip the step.
