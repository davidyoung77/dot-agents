---
name: commit
description: |
  Prepare a Conventional Commit scoped with the active work item ID when available
  before running git commit.
  Use when the user says "commit", "ready to commit", or wants to commit staged changes.
disable-model-invocation: true
---

# Commit

Prepare a commit that complies with Conventional Commit standards.

## Steps

1. Run `git status --short` and clearly show staged versus unstaged files.
2. If any unstaged or untracked files exist, notify the user and stop; never add files automatically.
3. Summarize the staged diff (only files already staged) so the user can verify what will be committed.
4. **Context Extraction**:
   - Attempt to extract a work item ref from the current branch name (for example `PROJ-123` or `WO-123`).
   - Attempt to infer the component scope from the staged file paths.
5. **Message Preparation**:
   - If a message was provided in `$ARGUMENTS`, validate it.
   - Ensure the format is `type(scope): description`.
   - Ensure `scope` includes the component and work item ref if present (e.g., `api, PROJ-123` or `api, WO-123`).
   - If no work item ref is found in the branch, use just the component scope.
   - If the format is incorrect or missing, propose a corrected version based on the diff and extracted context.
6. **Confirmation**: Display the final commit message and ask the user for explicit confirmation. Stop and wait for their response.
7. When the user approves, run `git commit` with the exact approved message.
8. After committing, run `git status` and report the result.
