---
name: jira-implement
description: |
  Jira story/bug/task implementation workflow. Use when user mentions a Jira ticket ID (e.g., PROJ-123) 
  and wants to implement it. Fetches issue details, creates implementation plan, waits for 
  approval, then implements with proper git workflow. Uses /commit and /pr skills for final steps.
---

# Jira Story Implementation

You implement Jira stories following a structured workflow with user checkpoints.

## Workflow Overview

When given a Jira ticket ID (e.g., PROJ-123), follow this workflow:

### Phase 1: Story Analysis & Validation
1. Fetch the Jira issue details using `atlassian___getJiraIssue`
2. Extract: summary, description, acceptance criteria, linked issues
3. **Validate story relevance** (especially for older stories or infrastructure changes):
   - Search codebase for key components mentioned in the story
   - Search for recent PRs that may have affected the component:
     ```bash
     gh pr list --state merged --search "component-name" --limit 10
     ```
   - Check git history for recent changes to affected files:
     ```bash
     git log --oneline --since="3 months ago" -- path/to/affected/files
     ```
   - If the story involves deprecation/removal, verify the component is still in use
   - If recent changes conflict with story assumptions, **flag this to the user**
4. Present a clear summary including:
   - Story requirements
   - Current state of affected components
   - Any recent changes that may impact implementation approach
   - **Recommendation**: Proceed as-is, adjust scope, or clarify with stakeholders

### Phase 2: Implementation Planning
1. Analyze the codebase to understand affected areas
2. Create a detailed implementation plan with:
   - Files to create/modify
   - Key changes needed
   - Testing approach (unit tests, API tests, UI verification)
   - Potential risks or dependencies

3. **STOP AND ASK:**
   > "Here is my implementation plan. Do you approve? (y/n)"
   
   **Wait for explicit user approval before proceeding to Phase 3.**

### Phase 3: Implementation (only after user approval)
1. **Verify/create feature branch** (mandatory before any code changes):
   - Check current branch: `git branch --show-current`
   - If already on a feature branch for this story with correct naming, proceed
   - Otherwise, ask user for their initials, then create from the default branch:
     ```
     git fetch origin
     git checkout -b feature/<initials>/<story-id>/<short-description> origin/<default-branch>
     ```
   - If existing branch has wrong naming, rename it: `git branch -m <old-name> <new-name>`
2. Create a todo list to track progress
3. Implement changes following existing code patterns and conventions
4. Add appropriate tests
5. Run linting and type checks as you go

### Phase 4: Testing & Verification
1. **Discover test commands**: Check for Makefile targets, package.json scripts, or service-specific test runners
2. **Run tests**: Execute lint, typecheck, and unit tests
3. **Verification based on change type**:

   **For API changes:**
   - Use Playwright to navigate to the app and capture an authenticated session
   - Extract JWT from request headers via `playwright___browser_network_requests`
   - Use curl with the captured JWT to test API endpoints
   - Take screenshots of successful responses as proof
   
   **For UI changes:**
   - Use Playwright to navigate to affected pages
   - Take screenshots of the changes
   - Verify interactions work correctly

4. **STOP AND ASK:**
   > "Here is my proposed verification approach. Do you approve? (y/n)"
   
   **Wait for user approval before executing verification.**

### Phase 4b: Parallel Code Review
After tests pass, dispatch parallel review droids to review the changes:

1. Launch `code-reviewer-opus` and `code-reviewer-gemini` (or `code-reviewer-gpt`) simultaneously via the Task tool
2. Provide both reviewers with: the Jira context, a summary of changes, and instructions to run `git diff` to see the exact diff
3. Wait for both reviews to return
4. Address any **Must Fix** findings before proceeding
5. Note **Observations** but do not act on them unless user requests

### Phase 5: Commit Preparation
1. Show the user what changed with `git diff`
2. **STOP AND PROMPT:**
   > "Ready to commit. Run `/commit` when you're satisfied with the changes."

3. **Wait for user to run `/commit`** - do NOT proceed until they confirm commit is done.

### Phase 6: PR Creation
**IMPORTANT: Do NOT create PR directly with `gh pr create`. Always defer to the `/pr` skill.**

1. **STOP AND PROMPT:**
   > "Commit complete. Ready to create PR. Run `/pr` when ready."

2. **Wait for user to run `/pr`**
3. If user says "proceed" or "create PR", remind them to use `/pr` - do not bypass this step

### Phase 7: Jira Update (Optional)
1. After PR is created, propose a Jira comment like:
   ```
   PR created: [PR_URL]
   
   Changes:
   - [Brief summary of changes]
   ```
2. **Ask user:** "Would you like me to add this comment to the Jira ticket?"
3. If approved, use `atlassian___addCommentToJiraIssue`

## Key Principles

- **Validate before planning** - Stories can become stale; verify current state matches assumptions
- **Always wait for approval** before implementing (after Phase 2)
- **Always wait for approval** before verification steps (Phase 4)
- **Never auto-commit or auto-push** - user controls git operations via `/commit` and `/pr` skills
- **Follow existing code patterns** - check codebase conventions first
- **Run all tests** before declaring implementation complete
- **Take screenshots** as evidence for PR

## Git Workflow Conventions

### Branch Naming
Feature branches follow: `feature/<initials>/<story-id>/<description>`
- Example: `feature/dy/PROJ-123/add-feature-flags`

Discover the default branch from `git symbolic-ref refs/remotes/origin/HEAD` or ask the user.

### Commit Messages
Use [Conventional Commits](https://www.conventionalcommits.org/en/v1.0.0/) format:
```
type(scope): description
```

**Types:** feat, fix, docs, style, refactor, test, chore

**Scopes:** Include the Jira ticket ID and infer the component from file paths:
- Example: `feat(PROJ-123, api): add new endpoint for feature flags`
- Example: `fix(PROJ-123, web): resolve navigation bug`

### PR Merge Strategy
- Feature branches -> **Squash and merge** into default branch
- Release/hotfix branches -> **Merge commit** (no squash)

## Codebase Discovery

At the start of implementation, orient yourself:
- Read project AGENTS.md for conventions
- Check for monorepo structure (nx, turborepo, lerna, etc.)
- Identify test runners and lint tools from package.json/Makefile/pyproject.toml
- Find PR templates in `.github/PULL_REQUEST_TEMPLATE.md`

## Environment URLs

**Default to local environment for testing.** Look for local dev setup docs (e.g., README, LOCAL_DEVELOPMENT.md, docker-compose.yml) to find the correct URLs.
