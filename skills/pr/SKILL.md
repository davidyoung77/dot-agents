---
name: pr
description: |
  Create a pull request with pre-filled template from branch and Jira context.
  Use when the user says "create PR", "open a PR", or is ready to push changes.
disable-model-invocation: true
---

# PR

Create a pull request for the current branch.

## 1. Gather Context

Run these commands in parallel:
- `git status` - verify clean working tree or staged changes
- `git log origin/develop..HEAD --oneline` - see commits to include
- `git diff origin/develop --stat` - see files changed
- `git rev-parse --abbrev-ref HEAD` - get current branch name

## 2. Extract Context

- Parse the Jira ID from the branch name (e.g., `feature/PROJ-123/description` -> `PROJ-123`)
- If a Jira ID is found, fetch the issue details for context (summary, acceptance criteria)
- If no Jira ID found, ask the user for context

## 3. Determine Base Branch

- If `$ARGUMENTS` contains a branch name, use that as the base
- Otherwise, default to `develop` (the repo's default branch)

## 4. Prepare PR Content

Read `.github/PULL_REQUEST_TEMPLATE.md` and fill it out:

### Description
- Summarize what changed based on commits and diff
- Reference the Jira story context (if available)

### JIRA Link
- Format: `https://<instance>.atlassian.net/browse/<TICKET-ID>` (if Jira ID found)
- Otherwise omit or mark N/A

### Author Checklist
- Pre-check items based on what you observe:
  - GitFlow: Feature branch -> suggest squash merge
  - Scope: Infer from file paths (infra, frontend, backend, etc.)
  - Leave testing checks unchecked for user to verify

### Test Steps
- Provide clear steps based on the changes made
- Include any API endpoints to test or UI flows to verify

### Screenshots
- If screenshots were taken during verification, list them for attachment
- Note: "Screenshots available in [location]" if applicable

## 5. Show PR Preview

Display the complete PR body in a code block and ask:
"Here's the proposed PR. Would you like me to create it, or would you like to make changes?"

**STOP and wait for user approval.**

## 6. Create the PR

Once approved, run:
```bash
gh pr create --draft --base [base-branch] --title "[PR title from first commit]" --body "[approved body]"
```

## 7. Report Success

- Display the PR URL
- Ask: "Would you like me to add a comment to the Jira ticket with the PR link?"

## Important Notes

- Never push without the user seeing the PR content first
- If `gh` CLI is not authenticated, provide instructions: `gh auth login`
- If there are uncommitted changes, warn the user and suggest running `/commit` first
- Keep PR title consistent with first commit message (conventional commit format)
- If no PR template exists in the repo, use a sensible default (description, test steps, checklist)
