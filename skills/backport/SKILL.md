---
name: backport
description: |
  Backport a release/hotfix branch into a base branch (default: develop). Creates a backport branch,
  handles merge, and guides through conflict resolution using resolve-conflicts skill. Uses the `commit`
  and `pr-create` skills for final steps.
---

# Backport Workflow

You are a git workflow specialist helping backport changes from release/hotfix branches into base branches.

## Phase 0: Update Existing Backport Branch

**If the user asks to "update" or "bring changes into" an existing backport branch:**

This handles the case where the source branch (e.g., `release/2.4.0`) has received new commits
after the backport branch was already created and a PR is open.

1. Fetch latest: `git fetch origin`
2. Checkout the backport branch (or navigate to its worktree)
3. Pull any remote updates: `git pull`
4. Merge the source branch: `git merge origin/<source-branch> --no-commit`
5. If conflicts, follow Phase 3 conflict resolution flow
6. If clean, commit with: `chore(backport): merge latest <source-branch> into backport branch`
7. Push -- the open PR will automatically update

**Skip to this flow when an existing backport branch is detected and user wants to update it.**

## Phase 1: Gather Information (STOP AND WAIT after each question)

**CRITICAL: Do NOT proceed until you have explicit answers from the user for ALL of the following. Ask ONE question at a time and WAIT for their response.**

1. **Source branch** - If not provided, ask:
   > "Which branch are you backporting FROM? (e.g., `release/1.2.3`)"
   
   **STOP and wait for response.**

2. **Base branch** - Ask to confirm even if they specified one:
   > "Base branch will be `develop`. Is that correct? (y/n)"
   
   **STOP and wait for response.**

3. **User initials** - Auto-detect from existing branches:
   - Run `git branch --show-current` and extract initials from patterns like `feature/<initials>/...` or `backport/<initials>/...`
   - If detected, use them automatically without asking
   - If not detected, ask: "What are your initials for the branch name? (e.g., `dy`)" and wait for response

4. **Worktree preference** - Ask:
   > "Would you like to use a git worktree? This keeps your current working directory untouched. (y/n)"
   
   **STOP and wait for response.**

5. **Confirm branch name** - After gathering all info, show the branch name and ask for confirmation:
   > "Branch name will be: `backport/<initials>/<source>-to-<base>-<mmddyyyy>`
   > 
   > Proceed? (y/n)"
   
   **STOP and wait for response.**

**Only after receiving confirmation on the branch name should you proceed to Phase 2.**

## Phase 2: Branch Setup

1. **Fetch latest**:
   ```bash
   git fetch origin
   ```

2. **Always create a backport branch** -- never PR directly from the source branch.
   CI pipelines often trigger based on the source branch name (e.g., `release/*`), causing
   unintended deployments. A dedicated backport branch avoids this.

3. **Generate branch name** using format:
   ```
   backport/<initials>/<source-branch>-to-<base-branch>-<mmddyyyy>
   ```
   - Date format: MMDDYYYY (e.g., 01212026)
   - Example: `backport/dy/release-1.2.3-to-develop-01212026`

4. **Verify you are in the correct repository** before creating branches/worktrees:
   ```bash
   git remote get-url origin
   ```
   Confirm the remote URL matches the expected repository. If the user is in a different repo
   (e.g., a monorepo sibling), warn them and ask to confirm before proceeding.

5. **Do a dry-run merge check** to inform the user about expected conflicts:
   ```bash
   git merge-tree --write-tree origin/<base-branch> origin/<source-branch> 2>&1
   ```
   - If exit code 0: "No conflicts expected -- clean merge."
   - If exit code non-zero or output contains "CONFLICT": "Conflicts expected in N files."
   - This is informational only -- still proceed with the user's chosen worktree/branch strategy.

6. **Create the backport branch:**
   - **If using worktree**:
     ```bash
     git worktree add -b <backport-branch> ../<backport-branch> origin/<base-branch>
     cd ../<backport-branch>
     ```
   - **If NOT using worktree**:
     ```bash
     git checkout -b <backport-branch> origin/<base-branch>
     ```

5. **Confirm setup** by showing:
   - Current branch: `git branch --show-current`
   - Current directory: `pwd`

## Phase 3: Merge Attempt

1. **Attempt the merge**:
   ```bash
   git merge origin/<source-branch> --no-commit
   ```
   - Using `--no-commit` to allow review before committing

2. **Check for conflicts**:
   ```bash
   git status
   ```

3. **If NO conflicts**:
   - Show `git diff --staged` summary
   - Proceed to Phase 4

4. **If conflicts exist**:
   - List all conflicted files clearly
   - **STOP and instruct user**:
     ```
     Merge conflicts detected in the following files:
     - [list files]
     
     The resolve-conflicts skill can help resolve these. Say "resolve conflicts" to proceed.
     Once resolved, return here and tell me "conflicts resolved" to continue.
     ```
   - **Wait for user to confirm conflicts are resolved**

## Phase 4: Post-Resolution Verification

1. **Verify resolution**:
   ```bash
   git status
   ```
   - Ensure no remaining conflicts
   - Show what files are staged/modified

2. **Show changes summary**:
   ```bash
   git diff --staged --stat
   ```

3. **STOP AND PROMPT:**
   > "Changes are ready. Please review with `git diff --staged` if needed.
   > 
   > For merge commits, use `git commit` directly (not the `commit` skill) to avoid pre-commit hook issues with large merges."
   
   **Recommended commit message format:**
   ```
   chore(backport): merge <source-branch> into <base-branch>
   
   Resolved conflicts in:
   - <file1> (<brief description of resolution>)
   - <file2> (<brief description of resolution>)
   ```

4. **Wait for user to confirm commit**

## Phase 5: Push and PR

1. **After commit confirmed**, compile the PR reviewer list from conflict authors tracked during resolution:
   - Map each author to the files they touched
   - Format: `Author Name - files: [file1, file2]`

2. **Track all conflict resolutions** for the PR description:
   - Maintain a table of ALL conflicted files and their resolutions
   - Format: `| file | resolution (current/incoming/manual + brief description) |`

3. **ALWAYS check for repo PR template BEFORE creating the PR body**:
   ```bash
   cat .github/PULL_REQUEST_TEMPLATE.md 2>/dev/null || echo "No template found"
   ```
   **CRITICAL**: If a template exists, use its EXACT structure as the foundation for the PR body.
   Fill in each section appropriately for a backport. Do not use a custom format that ignores the template.

4. **Create PR body file** using the repo's PR template as the base:
   - If a template exists, copy its structure and fill in each section:
     - `## Description`: Backport summary + conflict resolutions table (if any)
     - `## JIRA Link`: N/A - Backport merge
     - `## Git Workflow`: Check the backport/release checkbox, uncheck feature branch
     - `## Change Flags`: Check relevant items (e.g., Infrastructure Change if helm/k8s files changed)
     - `## Test Steps`: Review instructions + "Do NOT squash merge" warning
     - Author/Reviewer checklists: Check applicable items, mark N/A items as complete
   - If NO template exists, use this fallback structure:

   ```markdown
   ## Description
   Backport of <source-branch> into <base-branch>.

   **Resolved conflicts:**
   | File | Resolution |
   |------|------------|
   | `file1` | current/incoming/manual (description) |

   **Reviewers by Conflict Area:**
   - Author1: file1, file2
   - Author2: file3

   ## JIRA Link
   N/A - Backport merge

   ## Git Workflow
   - [ ] Feature branch: may be squashed and merged
   - [x] Backport, release or hotfix branch: The PR MUST NOT be squashed, just merged

   ## Test Steps
   1. Review conflict resolutions above
   2. Verify no regression in affected areas
   3. **Important**: Do NOT squash merge - use regular merge
   ```

5. **Create PR**:
   ```bash
   gh pr create --draft --base <base-branch> --title "chore(backport): merge <source> into <base>" --body-file /tmp/pr-body.md
   ```

6. **If `gh pr edit` fails due to Projects Classic deprecation**, the PR is still created - just inform user to manually edit if needed.

7. **Wait for user to confirm PR creation**

## Phase 6: Cleanup (if worktree was used)

1. **After PR is created**, remind user about worktree cleanup:
   ```
   Remember: You're in a worktree at `../<backport-branch>`.
   
   After the PR is merged, clean up with:
   1. cd <original-directory>
   2. git worktree remove ../<backport-branch>
   ```

## Key Principles

- **Always confirm** base branch with user before proceeding
- **Never auto-commit** - always defer to the `commit` skill when a normal commit workflow is appropriate
- **Never auto-create PR** - always defer to the `pr-create` skill
- **Pause for conflicts** - guide user to resolve-conflicts skill
- **Clear communication** at each phase about what's happening and what's next

## Example Invocation

User: "Backport release/1.2.3 to develop"

Response flow:
1. "Base branch will be develop. Correct?" → user: "yes"
2. (Auto-detects initials `dy` from current branch `feature/dy/...`)
3. "Use git worktree? (y/n)" → user: "n"
4. "Branch name will be: `backport/dy/release-1.2.3-to-develop-01212026`. Proceed? (y/n)" → user: "yes"
5. Creates branch, attempts merge, handles conflicts or proceeds
6. Guides through `commit` and `pr-create`
