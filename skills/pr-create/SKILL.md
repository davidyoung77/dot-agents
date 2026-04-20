---
name: pr-create
description: |
  Create a pull request with clean, concise template content, author-side validation,
  and evidence capture from branch and work item context. Use when the user says
  "create PR", "open a PR", or is ready to publish the current branch.
---

# PR Create

Create a pull request for the current branch.

This is the author-side readiness workflow:
- verify the branch is ready to publish
- run the validation that should happen before opening the PR
- capture evidence and remaining manual follow-up clearly
- capture a PDLC-aligned audit trail in the PR body
- keep the PR body clean and concise
- show the PR body before creating anything

Independent re-validation belongs to the `pr-review` skill.

## 1. Resolve Base Branch and Gather Context

1. Determine the base branch:
- If `$ARGUMENTS` contains a branch name, use it.
- Otherwise resolve the remote default branch with:
  ```bash
  git symbolic-ref --short refs/remotes/origin/HEAD | sed 's@^origin/@@'
  ```
- If that fails, ask the user.

2. Once the base branch is known, run these commands in parallel:
- `git status`
- `git log origin/<base-branch>..HEAD --oneline`
- `git diff origin/<base-branch> --stat`
- `git rev-parse --abbrev-ref HEAD`
- `git rev-parse --abbrev-ref --symbolic-full-name @{u}` (if the branch tracks a remote)

3. If there are uncommitted changes, stop and tell the user to use the `commit` skill first.

## 2. Extract Context

- Parse work item references only from explicit sources:
  - the branch name (for example `feature/PROJ-123/description` or `feature/WO-123/description`)
  - work item refs the user explicitly provided for the current task in this conversation
  - work item refs explicitly present in current-branch commit subjects
- Do not infer work item refs from unrelated chat history, vague prior context, or examples that were not clearly given as the source of truth for this PR
- Treat branch-matched refs and explicit user-provided refs as in-scope candidates for the PR body; use the precedence rules below to choose a primary work item when one exists
- If no user-provided or branch-matched ref exists, treat a single consistent commit-subject-discovered ref as an in-scope candidate only when the current-branch commit subjects consistently point to the same work item and do not look like merge, cherry-pick, or revert noise
- Otherwise treat commit-subject-discovered refs as secondary candidates; include or backlink them only when they clearly represent the same tracked work for this PR or the user confirms they should be included
- If a Jira ID is found, fetch the issue details and sharable link for context
- If a Jira lookup fails, keep the raw Jira identifier in plain text and do not invent a link or title you cannot confirm
- If an 8090 work order reference is found, fetch the work order details only when the specific 8090 project for this PR's work is confirmed by explicit user input or an earlier explicit project-resolution step in this workflow
- Include a sharable 8090 link only when the active tool explicitly exposes one; do not synthesize or guess 8090 URLs
- If the 8090 project is confirmed but no sharable link is exposed, use a plain-text fallback like `project-name / WO-123`
- If an 8090 work order reference is found but the specific project is not confirmed, ask the user which 8090 project it belongs to before attempting project-specific lookup, backlinking, or including a plain-text WO reference in the PR body
- If an 8090 lookup fails after the project is confirmed, keep a plain-text fallback like `project-name / WO-123` and do not invent a title or link
- If other explicit work item refs are provided, include them in the PR body only when the active tool exposes a confirmed sharable link/details, the identifier is unambiguous to reviewers, or the user explicitly wants the plain-text ref included
- If multiple in-scope work item references are found, keep them all and render them together in the PR body
- If multiple work item references are found, use this precedence for the primary work item context in the PR description:
  - a user-designated primary item, or otherwise the single explicit work item the user provided for this PR in the current task
  - otherwise the single branch-matched item
  - otherwise the single consistent commit-subject-discovered ref when no conflicting user-provided or branch-matched ref exists
  - otherwise ask the user which work item should be primary before naming one in the PR description
- If at least one confirmed or unambiguous work item ref exists, include the `Work Item References` section
- If no work item reference can be resolved and the repo template treats refs as optional and the user has not mentioned a work item for this PR, omit the section
- Otherwise ask the user before omitting it
- If branch naming, commit subjects, or the user request suggest a work item probably exists but it could not be resolved cleanly, ask the user before omitting it

## 3. Validation Readiness

This skill owns author-side validation before the PR is opened.

1. Discover local validation commands from the repo:
- `package.json` scripts such as `lint`, `typecheck`, `build`, `test`, `test:e2e`
- `Makefile` targets
- `pyproject.toml`, `go test`, or other project-native test runners

2. Run the lightweight local checks that are expected before opening the PR:
- lint
- typecheck
- build
- automated tests

3. Decide whether runtime validation is warranted. It usually is when the branch changes:
- user-visible UI flows
- API endpoints, webhooks, auth, session, or permissions
- integration boundaries, runtime configuration, or stateful workflows
- other critical paths where unit tests are not enough

4. If runtime validation is warranted and would require launching services, using browser automation, or making credentialed API calls, show a short proposed validation plan and ask for approval before running it.

5. If approved, run the least-cost real validation path available:
- existing smoke or e2e command if the repo already has one
- targeted browser validation for changed UI flows
- real API requests for changed endpoints or auth flows
- local dev / Docker Compose / documented non-prod service checks for integration-sensitive changes

6. Capture validation evidence clearly:
- commands run
- what passed, failed, or was skipped
- screenshots or evidence paths
- API endpoints exercised and status codes
- blockers and any remaining manual verification the reviewer should do

Evidence defaults:
- If the branch changes user-visible UI behavior, capture screenshots by default.
- Prefer before/after screenshots for behavior changes or bug fixes; for wholly new UI, an after screenshot is acceptable if there is no meaningful before state.
- If screenshots were not captured for a UI change, explain why in the PR body instead of silently omitting them.
- If the branch changes API behavior, capture concrete API evidence by default.
- API evidence should include the endpoint or route, method, status code, and the key response shape or behavior that was validated.
- If auth, permissions, or failure handling changed, try to capture at least one representative non-happy-path result too; if not possible, call that gap out explicitly.

If runtime validation was warranted but not run, say so explicitly and carry that gap into the PR body.

## 3b. Audit Trail and Traceability

Build a concise author-side audit trail aligned with PDLC traceability:
- `requirement/work item/blueprint/spec -> commit -> PR -> validation result`
- capture explicit references when available:
  - work item IDs or keys (Jira, 8090, etc.)
  - blueprint/spec/requirements docs
  - commits included in the PR
  - validation commands, evidence, and blockers
- record drift status:
  - no known requirements/blueprint drift
  - intentional drift with explanation and follow-up
  - suspected drift that `pr-review` should verify
- record what still needs reviewer attention or manual verification
- capture model, token, and cost usage for agent sessions used to prepare the PR only when the active tool or structured run output exposes exact values; never estimate or back-calculate it

### Usage Capture Guidance

- For headless scripted runs, prefer structured JSON output and normalize a canonical usage block into a one-line summary with `~/.agents/bin/extract-agent-usage`.
- The helper expects either a top-level exact usage object or a canonical nested `usage` block.
- Cursor headless example: `cursor-agent --print --output-format json "<prompt>" | ~/.agents/bin/extract-agent-usage`
- Factory / Droid headless runs: if the harness or mission exports structured JSON with exact usage fields, pass that JSON through `~/.agents/bin/extract-agent-usage` or copy the exact exposed values directly.
- Use the helper on a single run export or on a single exact rolled-up usage block already exposed by the harness.
- If the workflow spans multiple runs, record usage per run unless the harness already exposes one exact rolled-up total block.
- If the export contains multiple competing usage blocks or no canonical exact usage block, record `not available` rather than guessing.
- If the active tool or run does not expose exact usage or cost, record `not available`.

This audit trail should live in the PR body, not only in temporary local notes.

## 4. Prepare PR Content

Read `.github/PULL_REQUEST_TEMPLATE.md` and fill it out.

Keep the PR body clean:
- prefer the repo's existing template structure over inventing new sections
- summarize results instead of pasting raw command output, logs, stack traces, or API payloads
- keep descriptions and audit notes concise
- if a section has nothing meaningful to say, mark it clearly and move on

### Description
- Summarize what changed based on commits and diff
- Reference the primary work item context if available
- Keep it to one short paragraph or a few bullets, not a changelog

### Work Item References
- Include Jira and 8090 sharable links when available
- For other trackers or plain explicit refs, include them only when the user explicitly wants them, the identifier is unambiguous to reviewers, or a confirmed link/details are available
- If a sharable link is not available, include a plain-text work item reference only when the identifier is unambiguous for reviewers or the user explicitly wants it shown as-is
- If multiple work item references exist, list all of them
- If no work item references exist, omit the section or mark it `N/A` only when the repo template requires a placeholder

### Validation Results
- If the repo template already has a testing/verification section, use that instead of inventing a second one
- Prefer a short check/result summary over prose
- Include only the checks actually run in this session plus any clearly blocked items

### Author Checklist
- Check items only when they were actually verified in this run
- Do not infer or auto-complete testing items that were not executed

### Test Steps
- Describe what was already validated in this run
- List any API endpoints, UI flows, or smoke paths the reviewer should still verify manually
- If validation was blocked, state the blocker directly
- Keep this to reviewer-facing next steps, not a dump of everything you already did

### Screenshots / Evidence
- For user-visible UI changes, screenshots are expected by default; list before/after artifacts when they exist, or explain why screenshots were not captured
- For API changes, list the exercised endpoints, methods, status codes, and any saved evidence artifacts or notes
- If screenshots or other validation artifacts were captured, list them for attachment or reference
- If no artifacts were captured, say so plainly
- Reference artifact names or paths only; do not paste raw evidence into the PR body

### Audit Trail / Traceability
- Include the work-item references (Jira, 8090, etc.), blueprint, or spec references that explain why the change exists
- Summarize commits in scope when helpful
- Summarize validation evidence, skipped checks, and blockers
- Note the current drift assessment (`none known`, `intentional`, or `suspected`)
- Include token/cost usage when available from the tool or harness; otherwise mark it `not available`
- Keep this section to short bullets; do not turn the PR body into a long audit document

If the template has no natural place for this information, add a short `Audit Trail` section rather than dropping it.

If the repo has no PR template, use this clean default body shape:
- description
- work item references
- validation results
- test steps
- screenshots / evidence
- audit trail
- checklist

Preferred default shape:

```markdown
## Description
- Short summary of what changed and why

## Work Item References
- PROJ-123

## Validation Results
| Check | Result |
|-------|--------|
| Lint | ✅ Passed |
| Typecheck | ✅ Passed |
| Build | ⏭️ Not run |
| Automated Tests | ✅ Passed |
| Manual Runtime Validation | ⏭️ Reviewer should verify UI flow X |

## Test Steps
- Reviewer follow-up or remaining manual checks

## Screenshots / Evidence
- Screenshot names or evidence paths

## Audit Trail
- Work item/spec refs
- Drift status
- Key blockers or remaining reviewer attention
- Token/cost usage: `not available`

## Checklist
- Template or repo checklist items
```

## 5. Show PR Preview

Display the complete PR body in a code block and ask:

`Here's the proposed PR. Would you like me to create it, or would you like to make changes?`

**STOP and wait for user approval.**

## 6. Create the PR

Once approved:

1. If the branch has no upstream, ask before pushing it with:
```bash
git push -u origin HEAD
```

2. Create the PR:
```bash
gh pr create --draft --base <base-branch> --title "<title>" --body "<approved body>"
```

Use the first commit message or the user's preferred title as the PR title. Keep it consistent with the branch's conventional-commit style when possible.

## 7. Report Success

- Display the PR URL
- Summarize the validation evidence and short audit-trail details included in the PR body
- Ask: `Would you like me to add the PR link back to the linked work item(s) where the tool supports it?`

## Important Notes

- Never create the PR before the user sees and approves the PR content
- If `gh` CLI is not authenticated, provide instructions: `gh auth login`
- If there are uncommitted changes, stop and direct the user to the `commit` skill first
- Use the repo's actual default branch unless the user overrides it
- Validation should be honest: if something was not run, leave it unchecked and say so
- When runtime validation is blocked, record the blocker rather than pretending the branch is fully verified
- Keep the PR body clean and traceable: include drift status and exact token/cost usage only when the active tool exposes it, otherwise use `not available` without burying the useful summary in audit detail
