---
name: pr-review
description: |
  Review a GitHub pull request with full local validation and multi-model consensus.
  Use when user shares a PR URL, says "review this PR", or asks to review a pull request.
  Handles enterprise SSO repos (uses gh CLI, not FetchUrl). Runs static validation,
  executes tests, dispatches to 6 reviewer droids in parallel, consolidates findings,
  and posts inline review comments on the correct diff lines.
---

# PR Review

End-to-end pull request review: fetch, validate locally, multi-model review, present findings,
then post inline comments where they belong.

## Mode Detection

| User says | Action |
|---|---|
| "review this PR", "review PR #N", shares a PR URL | Full review flow |
| "just run the tests on this PR" | Skip droid dispatch, just validate locally |
| "post the review" | Post previously presented findings |

## Flow

### 1. Fetch PR Details

Use `gh` CLI (NOT FetchUrl -- enterprise repos hit SSO walls):

```bash
gh pr view <number> --json title,body,files,commits,headRefName,baseRefName,state,additions,deletions
gh pr diff <number>
```

Extract: branch name, file list, diff stats, linked tickets.

### 2. Checkout and Install

```bash
git fetch origin <headRefName>
git checkout <headRefName>
npm install  # or project-appropriate install
```

Preserve the user's working state -- stash if needed, restore after.

### 3. Static Validation

Detect and run the project's validation commands. Check for:
- `package.json` scripts: `typecheck`, `lint`, `build`, `bddgen`
- Common patterns: `tsc --noEmit`, `eslint`, `prettier --check`

Run ALL that exist. Report results.

### 4. Test Execution

Run the project's test suite. Capture:
- Total tests, passed, failed, skipped
- Failure messages with file/line references
- Whether failures are pre-existing (checkout base branch and test) or introduced by PR

If tests require auth/credentials and fail at auth step:
- Check for project-specific refresh commands (`.factory/commands/refresh-tokens.md`)
- Check for auth helper scripts
- Report the auth blocker clearly -- do NOT silently skip tests

### 5. Multi-Model Review (All 6 Droids)

Dispatch to ALL 6 reviewer droids in parallel:

**Spec reviewers** (architecture/design):
- `spec-reviewer-opus`
- `spec-reviewer-gpt`
- `spec-reviewer-gemini`

**Code reviewers** (implementation quality):
- `code-reviewer-opus`
- `code-reviewer-gpt`
- `code-reviewer-gemini`

Each gets the same prompt with:
- Full file list to read
- Test results from step 4
- Review criteria (anti-patterns, security, dead code, duplication, error handling)
- Instruction to return structured findings with severity and line references
- Instruction to NOT make edits

Handle timeouts gracefully -- if a droid times out, note it and continue with results from those that completed.

### 6. Consolidate Findings

Merge findings across all droids:
- **Consensus items**: flagged by 2+ droids -> higher confidence
- **Unique items**: flagged by only 1 droid -> still include but note
- Deduplicate (same finding on same line from multiple droids)
- Group by severity: Critical > High > Medium > Low
- Include positive findings too (good patterns, nice changes)

### 7. Present to User for Approval

Show the consolidated review BEFORE posting. Include:
- Test results summary table
- Findings grouped by severity with file:line references
- Which droids agreed on each finding
- Proposed review action (APPROVE, REQUEST_CHANGES, COMMENT)

Ask: "Ready to post this review, or want to adjust anything?"

### 8. Post Inline Review

Use the GitHub API to post a proper review with inline comments:

```bash
gh api repos/{owner}/{repo}/pulls/{number}/reviews --input review.json
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

## Important Notes

- **Never post without user approval.** Always present findings first.
- **Don't post a wall-of-text comment.** Use inline comments on specific lines.
- **Include test results** in the review body summary.
- **Note security findings prominently** -- hardcoded secrets, exposed tokens, etc.
- **Positive feedback matters** -- call out good patterns, not just problems.
- **Handle auth gracefully** -- if tests can't run due to expired tokens, say so clearly and check for project-specific refresh mechanisms.
- **Restore git state** -- if you checked out the PR branch, offer to switch back when done.
