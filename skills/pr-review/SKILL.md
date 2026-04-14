---
name: pr-review
description: |
  Review a GitHub pull request using existing PR checks, an initial reviewer pass,
  and targeted runtime validation when warranted.
  Use when user shares a PR URL, says "review this PR", or asks to review a pull request.
  Handles enterprise SSO repos (uses gh CLI, not FetchUrl), reviews in an isolated
  git worktree, reconciles prior feedback, inspects existing PR checks, dispatches code
  reviewers first, optionally adds spec/PRD reviewers when warranted, consolidates findings,
  and posts inline review comments on the correct diff lines.
---

# PR Review

End-to-end pull request review: resolve PR context, review in an isolated worktree,
inspect existing PR checks, run reviewer passes first, then targeted runtime
validation when warranted, collect feedback, present findings, and optionally
post inline comments where they belong.

## Mode Detection

| User says | Action |
|---|---|
| "review this PR", "review PR #N", shares a PR URL | Full review flow |
| "just run the tests on this PR" | Skip reviewer dispatch and posting; summarize existing PR checks first and rerun local checks only if the user explicitly asks or a repro is needed |
| "post the review" | Post previously approved findings from a temporary saved report/payload when available |

## Flow

### 0. Resolve Context and Review Artifacts

After resolving the PR number and repo context, use `gh` CLI (NOT FetchUrl -- enterprise repos hit SSO walls):

```bash
gh pr view <number> --json title,body,files,commits,headRefName,baseRefName,state,additions,deletions,statusCheckRollup
gh pr diff <number>
gh pr checks <number>
```

1. Determine the repository and PR number:
   - If the user shared a PR URL, use it directly or extract the PR number from it.
   - Otherwise try `gh pr view --json number --jq '.number'` from the current branch.
   - If neither works, ask the user for the PR number.
2. Determine the GitHub `owner/repo` slug from `git remote get-url origin`.
3. Determine local path variables for artifact and worktree paths:
   - `REPO_ROOT="$(git rev-parse --show-toplevel)"`
   - `REPO_NAME="$(basename "$REPO_ROOT")"`
4. Resolve artifact locations:
   - `WORKTREES_ROOT="${WORKTREES_ROOT:-$HOME/git/worktrees}"`
   - `WORKTREE_PATH="$WORKTREES_ROOT/$REPO_NAME/pr-<NUMBER>"`
   - Report directory: first look for agent output/work artifact guidance in `AGENTS.md`, `.agents/AGENTS.md`, `.factory/AGENTS.md`, or `.agent/AGENTS.md`
   - If the repo defines only a base artifact/work directory, nest PR review artifacts under `pr-reviews/pr-<NUMBER>/`
   - If no repo-specific artifact directory is defined, fall back to `<REPO_ROOT>/.agents/work/pr-reviews/pr-<NUMBER>/`
   - Only if there is no natural project-local home, fall back to `~/.agents/work/pr-reviews/<owner>/<repo>/pr-<NUMBER>/`
   - Set `REPORT_DIR` to the final resolved PR-specific directory before writing any review artifacts
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
   - PR check status / relevant CI job results
   - linked tickets if present
9. Capture the exact PR head SHA and `baseRefName` once.
   - Use that pinned SHA for payload posting and line validation.
   - Use the resolved base branch everywhere you need an affected-file or diff comparison; do not hardcode `origin/main`.
10. Print a short summary for the user:
   - PR title and number
   - base <- head @ pinned SHA
   - additions/deletions and changed-file count
   - PR check summary
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
git worktree add "$WORKTREE_PATH" "origin/<headRefName>" --detach
```

1. If the target worktree path already exists, remove and recreate it only if it is clearly disposable.
2. Do not install dependencies or start services yet unless a later review or runtime-validation step actually needs them.
3. If worktree creation fails, warn the user and ask before falling back to a diff-only review.
4. If the PR diff exceeds roughly 500 lines, save it to `<REPORT_DIR>/pr-<NUMBER>-diff.txt` and use that file path in reviewer prompts instead of inlining the full diff.
5. Keep the user's main checkout untouched throughout the review.

### 3. Summarize Existing PR Checks

Before rerunning anything locally, inspect the PR's existing GitHub checks.

- Capture the status of lint, typecheck, build, unit/integration tests, and any repo-specific verification jobs.
- Use GitHub check status as the default source of truth for those categories when the repo's CI is healthy.
- Present a compact validation table early so reviewers and the user can see what CI already covered.

Only rerun local static/test commands when:
- CI is missing, failing, or flaky and you need to disambiguate whether a problem is code or environment
- the repo's PR checks are known to be incomplete
- the user explicitly asks for local reruns
- a reviewer finding needs local reproduction

If you rerun static/test commands locally:
- keep them clearly labeled as local reproduction, not the primary validation path
- rerun only the minimum set needed for the question at hand

### 4. Reviewer Fan-Out First (Code Always, Spec/PRD Conditionally)

Dispatch the reviewer pass before any heavy runtime/browser/API validation.

Select the reviewer matrix before dispatching:

- If the platform supports distinct model assignment per reviewer, use the full reviewer family and treat agreement as true cross-model signal.
- If the platform does not support true per-reviewer model selection, the same family can still be useful as prompt-diverse or lens-diverse passes, but call that limitation out explicitly.

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

If the early reviewer pass finds obvious blocking defects or major unresolved prior feedback that make runtime validation low-value, stop and ask the user before spending more time on runtime/browser/API checks.

If optional reviewer families are skipped, note that explicitly in the final summary and report.

Each reviewer gets the same core context:
- PR number, title, URL, base branch, head branch, and pinned head SHA
- PR description/body
- absolute worktree path
- full changed-file list
- PR checks summary
- any local reproduction results already run
- prior unresolved feedback, if any
- the diff inline for small PRs, or the saved diff file path for large PRs
- note that targeted runtime validation is still pending unless it has already been run

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
- Include a one- or two-sentence confidence rationale tied to the actual evidence reviewed, blockers encountered, and coverage achieved
- Never make edits

Handle reviewer failures gracefully:
- If you have an explicit failure reason, use it verbatim or near-verbatim in the temp report and posted matrix, for example `provider error`, `auth error`, `tool failure`, or `no result returned`
- Use `timed out` only when the reviewer actually hit an explicit timeout signal
- Continue with results from the reviewers that completed when there is still enough signal

### 5. Runtime / Real-Surface Validation (When Warranted)

Decide whether independent runtime validation is warranted after the initial reviewer pass. It usually is when the PR changes:
- user-visible UI flows
- API endpoints, webhooks, auth, session, or permissions
- integration boundaries, runtime configuration, migrations, or other runtime-sensitive behavior
- critical paths the user explicitly wants smoke-tested

When warranted:
- install dependencies or start local services only now, when the review has already cleared the basic "worth validating further" bar
- prefer the least-cost real validation path available:
  - existing smoke or e2e command if the repo already provides one
  - browser-based validation for changed UI flows
  - real API requests for changed endpoints or auth flows
  - local dev / Docker Compose / documented non-prod service checks for integration-sensitive behavior
- read `AGENTS.md`, `.agents/AGENTS.md`, README, and project commands to find the correct runtime path before inventing one
- before launching services, using browser automation, or making credentialed requests, present a short proposed runtime validation plan and ask for approval unless the user already explicitly asked for it
- capture a runtime validation summary:
  - what was tested
  - what passed, failed, or was skipped
  - screenshots or evidence paths if produced
  - API endpoints exercised and status codes
  - blockers and follow-up manual verification needed

If runtime validation requires auth/credentials and blocks:
- check for project-specific refresh commands (`.agents/commands/refresh-tokens.md` or legacy `.factory/commands/refresh-tokens.md`)
- check for auth helper scripts
- report the auth blocker clearly -- do NOT silently skip it

If runtime validation was warranted but blocked, include that gap clearly in the final summary and in the temp report.

### 6. Consolidate Findings

Normalize reviewer-specific outputs into a common findings list, then merge:
- Code reviewer `must_fix` findings map to the highest severities
- Code reviewer `should_fix` findings map to medium/high severities
- Spec reviewer gaps and risks become findings with clear severity and file/context references where available
- PRD reviewer gaps, contradictions, and requirements-coverage risks become findings with clear severity and document context where available
- Suggestions and positive observations remain non-blocking items

Then consolidate:
- **Consensus items**: flagged by 2+ reviewers -> higher confidence
- **Unique items**: flagged by only 1 reviewer -> still include but note
- Deduplicate (same finding on same line from multiple reviewers)
- When multiple reviewers assess the same location differently, keep the majority assessment and note dissent
- Group by severity: Critical > High > Medium > Low
- Include positive findings too (good patterns, nice changes)

Also capture lightweight review metadata for the temp report and posted body:
- per-reviewer confidence percentages and rationales
- traceability refs found or missing
- blueprint/requirements drift conclusions when optional reviewer families ran
- PR check coverage plus any local reproduction and runtime coverage, skips, and blockers
- token/cost usage by reviewer or phase only when the active tool or structured run output exposes exact values
- any remaining evidence gaps the human reviewer should know

### Usage Capture Guidance

- For headless scripted runs, prefer structured JSON output and normalize a canonical usage block into a one-line summary with `~/.agents/bin/extract-agent-usage`.
- The helper expects either a top-level exact usage object or a canonical nested `usage` block.
- Cursor headless example: `cursor-agent --print --output-format json "<prompt>" | ~/.agents/bin/extract-agent-usage`
- Factory / Droid headless runs: if the harness or mission exports structured JSON with exact usage fields, pass that JSON through `~/.agents/bin/extract-agent-usage` or copy the exact exposed values directly.
- Use the helper on a single run/reviewer export or on a single exact rolled-up usage block already exposed by the harness.
- If the workflow spans multiple runs or reviewers, record usage per run/reviewer unless the harness already exposes one exact rolled-up total block.
- If the export contains multiple competing usage blocks or no canonical exact usage block, record `not available` rather than guessing.
- If the active tool or run does not expose exact usage or cost, record `not available`.

### 7. Write Temporary Recovery Artifacts

Before asking to post, write a markdown review report to `<REPORT_DIR>/pr-<NUMBER>-review.md`.

Include:
- PR metadata and URL
- PR check summary
- any local reproduction summary
- runtime validation summary
- prior-feedback summary
- reviewer matrix used, including any optional reviewer families that were skipped
- per-reviewer confidence percentages and rationales
- traceability and drift summary
- consolidated findings grouped by severity
- which reviewers agreed on each finding
- the proposed review action: `APPROVE`, `REQUEST_CHANGES`, or `COMMENT`

Save intermediate artifacts when useful:
- runtime validation evidence paths or notes, if produced
- raw reviewer outputs
- saved diff file for large PRs

The temp report is where the detailed evidence lives. Keep the posted review body cleaner and shorter.

If the user later says "post the review", reuse the temporary saved review report and payload when available instead of recomputing everything.

### 8. Present to User for Approval

Show the consolidated review BEFORE posting using the same top-level structure you plan to post:
- `## Code Review — PR #<NUMBER>`
- `### Validation Results`
- `### Reviewer Matrix`
- `### Summary`

Within that structure:
- Validation Results should distinguish existing `PR Checks` from `Manual Runtime Validation`
- Reviewer Matrix should keep each reviewer on one row; include confidence percentages inline only if they add clarity, for example `✅ APPROVE (92%)`
- When a reviewer fails, prefer precise status labels such as `⚠️ Provider error`, `⚠️ Failed`, or `⚠️ No result`; use `⚠️ Timed out` only for explicit timeouts
- Summary should stay to one short paragraph plus a short consensus line when helpful
- Keep prior-feedback detail, confidence rationales, and long audit-trail notes in the temp report unless they materially affect the verdict
- Prefer inline review comments for detailed findings; do not add extra top-level body sections unless there is a non-inline issue that truly needs calling out
- If traceability/drift/runtime blockers materially affect the verdict, mention them briefly inside Summary rather than turning the post into a long audit document
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
  "body": "Markdown review body following the clean review template below",
  "comments": [
    {
      "path": "relative/file/path.ts",
      "line": <line number in NEW file>,
      "body": "**Severity: Finding title**\n\nDetails and fix suggestion."
    }
  ]
}
```

Use this clean body shape unless the user explicitly asks for something else:

```markdown
## Code Review — PR #<NUMBER>

### Validation Results

| Check | Result |
|-------|--------|
| PR Checks | ✅ lint, typecheck, build, and tests passed |
| Manual Runtime Validation | ⏭️ Not warranted |

### Reviewer Matrix

| Reviewer | Result |
|----------|--------|
| code-reviewer-opus | ✅ APPROVE (92%) |
| code-reviewer-gpt | ✅ APPROVE (88%) |
| code-reviewer-gemini | 💬 COMMENT (76%) |
| spec-reviewers | ⏭️ Skipped (no blueprint drift risk) |
| prd-reviewers | ⏭️ Skipped (no requirements review needed) |

### Summary

Concise assessment here.

**Consensus: APPROVE (2/3)**
```

Format rules for the posted body:
- Keep this section order: title, validation results, reviewer matrix, summary
- Use `Code Review` in the title, not `Automated Review` or `Automated Review Summary`
- Validation Results should summarize existing PR checks first, then any manual runtime/browser/API validation
- Keep the Summary concise; if possible, fit in one short paragraph plus a consensus line
- Keep prior-feedback details, confidence rationales, screenshots, API payloads, and detailed audit-trail notes in the temp report unless they materially change the verdict
- Add extra top-level sections only when a non-inline finding truly needs to be called out outside the summary
- Preserve reviewer failure reasons in the Reviewer Matrix when known; do not collapse all missing reviewer results into `Timed out`

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
- **Use `WORKTREES_ROOT` when constructing worktree paths.** Default to `$HOME/git/worktrees` only when the environment variable is unset.
- **Run reviewers before heavy validation.** Use the early reviewer pass to decide whether runtime/browser/API validation is worth the time.
- **Don't duplicate healthy CI by default.** Prefer existing PR checks for lint/typecheck/build/test status; rerun locally only when the user asks or a repro is needed.
- **Don't post a wall-of-text comment.** Use inline comments on specific lines where possible.
- **Keep the review format clean.** Default to `Code Review` title plus `Validation Results`, `Reviewer Matrix`, and `Summary`.
- **Keep the review auditable.** The temp report should preserve the detailed traceability, drift, and exact usage/cost context when the active tool exposes it; otherwise use `not available` and promote only the highest-signal items into the posted body.
- **Be explicit about review diversity.** Distinguish true cross-model agreement from prompt-diverse agreement when the platform cannot assign different models per reviewer.
- **Include PR check results and manual runtime validation** in the review body summary.
- **Include reviewer confidence lightly.** Prefer inline percentages in the reviewer matrix over a separate confidence section.
- **Note security findings prominently** -- hardcoded secrets, exposed tokens, etc.
- **Positive feedback matters** -- call out good patterns, not just problems.
- **Handle auth gracefully** -- if runtime or repro checks can't run due to expired tokens, say so clearly and check for project-specific refresh mechanisms.
- **Use saved reports for follow-up.** If the user later says "post the review", reuse the latest approved findings/report when possible instead of recomputing everything.
- **Delete temp artifacts after success.** Local PR review files are for recovery, not archival, once the review is safely in the PR.

## Error Handling

- If `gh` is not authenticated, instruct the user to run `gh auth login`.
- If a reviewer fails, record the most specific reason available (`provider error`, `timed out`, `no result`, etc.), continue with the remaining results, and note the missing reviewer.
- If the worktree cannot be created, ask whether to continue with a diff-only review.
- If local reproduction or runtime validation fails due to environment/auth setup, report the blocker clearly and do not silently skip the step.
