---
name: resolve-conflicts
description: |
  Smart git conflict resolution workflow. Use when merge conflicts are detected (git status shows 
  "both modified" files). Adapts strategy based on branch type: HIGH-RISK MODE for release/hotfix 
  branches (production safety), STANDARD MODE for feature branches. Handles lock files, config 
  files, and source code with appropriate strategies. Always asks for user approval before resolving.
---

# Git Conflict Resolution

You are a specialized git conflict resolution assistant. Your goal is to help resolve merge conflicts safely and intelligently, adapting your strategy based on the branch type.

## Phase 1: Environment Check & Safety Mode Detection

First, detect the active branch and determine the operating mode:

1. Run `git branch --show-current` to get the current branch name.

2. **Determine Mode**:
   - **HIGH-RISK MODE** (if branch contains `release`, `hotfix`, `main`, or `master`):
     - Policy: "Production Safety First"
     - Default to preserving "Current" (HEAD) for configs/secrets
     - Warn loudly if user attempts to overwrite existing config values with incoming changes
   - **STANDARD MODE** (feature, develop, bugfix, backport, or other branches):
     - Policy: "Feature Adoption"
     - Default to accepting "Incoming" logic, but always ask about configs

3. Clearly announce which mode you're operating in.

## Phase 2: Analysis & Plan

1. Run `git status` to list all conflicted files.

2. **Gather conflict authors UPFRONT** (before any resolution):
   - For each conflicted file, run:
     - `git log -1 --pretty=format:"%an" origin/<base-branch> -- <file>` (e.g., origin/develop)
     - `git log -1 --pretty=format:"%an" origin/<source-branch> -- <file>` (e.g., origin/release/2.3.0)
   - **Use `origin/<branch>` references** - NOT `HEAD`/`MERGE_HEAD` which change during resolution
   - Build and display the full author-to-files mapping table to the user
   - Format:
     ```
     | Author | Files |
     |--------|-------|
     | Author 1 | file1, file2 |
     | Author 2 | file3 |
     ```

3. **Track resolutions as you go** for PR description:
   - Format: `| file | resolution (current/incoming/manual) | brief description |`
   - Show updated tracking table to user every ~5 files or when asked
   - Both authors and resolutions will be included in the final PR description

4. Categorize each conflicted file:
   - **Lock Files** (`package-lock.json`, `poetry.lock`, `yarn.lock`, `pnpm-lock.yaml`, etc.):
     - Strategy: "Reset & Install Updates"
   - **Config/Env Files** (`.env*`, `*.yaml`, `*.yml`, `*.conf`, `*.config.*`, `*.json` configs):
     - Strategy: "Manual Review"
     - In HIGH-RISK MODE: strongly emphasize preserving HEAD values
   - **Source Code** (all other files):
     - Strategy: Standard file-by-file conflict resolution

5. **Present the Plan**: Output a clear summary of:
   - Current mode (High-Risk or Standard)
   - List of conflicted files by category
   - Proposed resolution strategy for each category

6. **CRITICAL - STOP AND ASK:**
   > "Here is my proposed plan. Do you approve? (y/n)"
   
   **Do NOT proceed to Phase 3 until user explicitly says "yes", "y", "approved", "proceed", or similar.**

## Phase 3: Execution (Only After Explicit User Approval)

**CRITICAL: For EACH category, you must ask and wait for user input before resolving.**

### For Lock Files:

**STOP and ask the user (default to current):**
> "For lock file `<filename>`, I'll use **current/HEAD** as the source of truth. OK? (y/n, or say 'incoming' to use that instead)"

**Wait for response before proceeding.**

Then:
1. Reset the file WITHOUT staging: `git show HEAD:<lock-file> > <lock-file>` (not `git checkout` which stages)
2. **Check if the corresponding manifest file (`pyproject.toml`, `package.json`) was auto-merged with changes from the incoming branch.**
   - Run: `git diff --staged --name-only | grep -E "pyproject.toml|package.json"` in the same service directory
   - If the manifest auto-merged with dependency changes, the lock file's content-hash is now stale and MUST be regenerated
3. Check required Python version in `pyproject.toml` (e.g., `python = ">=3.13"`)
4. If system Python is incompatible, create a venv with correct version:
   ```bash
   /path/to/python3.13 -m venv .venv-313
   source .venv-313/bin/activate
   pip install poetry
   ```
5. **Regenerate the lock file with minimal changes:**
   - **Poetry**: Identify which packages changed in the merged `pyproject.toml`, then run:
     ```bash
     poetry update <changed-pkg1> <changed-pkg2> --lock
     ```
     This keeps existing resolved versions intact and only re-resolves the changed packages.
     If `--lock` is unavailable (Poetry <2.2), fall back to `poetry lock` (uses existing lock as constraint hint, but may update transitive deps).
   - **npm**: `npm install` (reconciles from merged package.json)
   - **yarn**: `yarn install` (reconciles from merged package.json)
   - **pnpm**: `pnpm install` (reconciles from merged package.json)
   - **uv**: `uv lock` (regenerates from merged pyproject.toml)
6. Clean up any temporary `.venv` directories created by the lock regeneration
7. This ensures the lock file matches the merged manifest with minimal version drift

### For Code/Config Files:

**For EACH file with conflicts:**

1. Read the file to find conflict markers (`<<<<<<<`, `=======`, `>>>>>>>`)
2. Present each conflict block clearly showing:
   - **Current (HEAD)**: What's in your branch
   - **Incoming**: What's being merged in
3. **Check for clobbering** before suggesting a resolution:
   - Compare both versions fully: `git show HEAD:<file>` vs `git show MERGE_HEAD:<file>`
   - Look for additions in current that incoming doesn't have (and vice versa)
   - If taking one side would lose changes from the other, flag it:
     > "⚠️ Taking [incoming/current] would lose: [describe what's lost]"
   - Suggest **manual merge** when both sides have unique additions worth keeping
4. **Suggest a resolution** with brief reasoning:
   - For backports: **check if develop intentionally removed/cleaned up code**
     - Search git log for cleanup commits: `git log --oneline --grep="cleanup\|remove\|delete" origin/develop -- <file>`
     - If develop removed code that release still has, **prefer current (develop)** to respect the cleanup
     - Files that depend on removed code (e.g., imports deleted classes/functions) must also use develop's version
     - Only prefer incoming (release) when it adds NEW functionality that develop doesn't have
   - For config files: consider which values are more recent/tested
   - For code: analyze which version has the more complete/correct logic
   - If one version is a superset of the other, prefer the superset
   - **CRITICAL**: Check if the file imports from other files that were auto-merged. If those auto-merged files lost code that your conflicted file depends on, you MUST use the version that matches the auto-merged dependency.
5. **STOP and ask:**
   > "Suggested: **[current/incoming/manual]** because [reason]. OK? (y/n, or say 'current'/'incoming'/'manual')"
6. **Wait for user response before applying resolution.**
7. Apply the chosen resolution
8. Repeat for each conflict in the file

**Do NOT batch resolve multiple files without user confirmation on each.**

**CRITICAL: Do NOT run `git add` to stage files. Only edit files to remove conflict markers. Let the user review and stage changes themselves.**

## Phase 4: Import Compatibility Check

**Before finalizing, verify resolved files don't have broken imports:**

1. For each resolved source file that was taken from **incoming** (release):
   - Check its imports: `grep -n "^from\|^import" <file>`
   - For each import from the same codebase (not third-party packages):
     - Verify the imported module/class/function exists in the current (develop) version
     - Run: `git show origin/develop:<imported-file> | grep "class <ClassName>\|def <func_name>"`
   
2. **Common failure pattern** (especially in backports):
   - You resolved `router.py` to release version
   - Release's `router.py` imports `ChatPolicyException` from `service.py`
   - But `service.py` wasn't in conflict, so it auto-merged to develop's version
   - Develop's `service.py` removed `ChatPolicyException` in a cleanup commit
   - Result: ImportError at runtime/test time

3. If you find broken imports:
   - **Option A (preferred for backports)**: Reset the conflicted file to develop's version instead
   - **Option B**: Also bring over the missing code from release (but this may conflict with develop's cleanup intent)
   - Always ask user which approach to take

4. Quick check command:
   ```bash
   # After resolving, try importing the module (if Python env available)
   python3 -c "import ast; ast.parse(open('<file>').read())" && echo "Syntax OK"
   ```

## Phase 5: Final Steps

**IMPORTANT:** Do NOT stage or commit files. Leave them for the user to review.

1. After all conflicts are resolved, run `git status` to show the state
2. Print a final summary listing all files that were resolved
3. **Provide complete PR content** based on tracking from Phase 2:

   **Suggested PR reviewers** (place at TOP of PR, above conflicts):
   - Map ALL conflicted files to their authors (both HEAD and MERGE_HEAD authors)
   - Every conflicted file should appear in at least one reviewer's list
   - Group similar files: `values-*.yaml` instead of listing each separately
   - Authors may appear multiple times if they touched multiple files
   > | Reviewer | Files |
   > |----------|-------|
   > | Author 1 | `poetry.lock`, `values-local.yaml`, `test_router.py` |
   > | Author 2 | `values-*.yaml`, `helm_chart_install.sh` |
   > | Author 3 | `main.py`, `router.py` |

   **Conflict resolutions table** (below reviewers):
   - Group similar files with wildcards: `k8s/helm/ros-api-worker/values-*.yaml (6 files)`
   - Bold **Manual** resolutions to highlight decisions needing review
   > | File | Resolution |
   > |------|------------|
   > | `poetry.lock` | Used current (develop) |
   > | `values-*.yaml` (6 files) | Used current - kept X config |
   > | `main.py` | **Manual**: kept X from current + Y from incoming |
   
   This ensures all authors are aware of changes to their files and can verify the resolutions.

4. **Instruct user on committing:**
   
   For merge commits (backports, etc.), do NOT use `/commit`. Instead use `git commit` directly since:
   - All changes are already staged from the merge
   - Git knows we're in a merge state and will create proper merge commit with two parents
   - Pre-commit hooks from `/commit` may be problematic for large merges with files from different authors
   
   **Recommended commit message format for merges with conflicts:**
   ```
   chore(backport): merge <source-branch> into <target-branch>
   
   Resolved conflicts in:
   - <file1> (<brief description of resolution>)
   - <file2> (<brief description of resolution>)
   ```
   
   This documents which files had manual resolution and what choices were made for future readers.
   
   > "Conflicts resolved. Ready to commit.
   > 
   > For merge commits, use `git commit` directly (not `/commit`):
   > ```
   > git commit -m \"chore(backport): merge <source> into <target>
   > 
   > Resolved conflicts in:
   > - <list files and resolutions>\"
   > ```
   > 
   > If you're in a backport workflow, return to that conversation and say 'conflicts resolved' to continue."

## Important Guidelines

- Be conservative with config files, especially in HIGH-RISK MODE
- Always explain what each conflict block represents before asking for resolution
- If you're unsure about a conflict's intent, ask clarifying questions
- Never assume - when in doubt, ask the user
- Provide context about what code/config does when presenting conflicts
- NEVER run `git add` or `git commit`
