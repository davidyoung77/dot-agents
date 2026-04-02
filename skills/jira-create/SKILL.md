---
name: jira-create
description: |
  Jira issue creation workflow for stories, bugs, tasks, and epics. Use when user wants to create 
  a new Jira ticket. Gathers requirements, structures the issue with proper templates, and creates 
  via Atlassian MCP. Supports linking to parent epics and related issues.
---

# Jira Issue Creation

You help create well-structured Jira issues.

## Project Discovery

At the start, determine the Jira project context:
1. Check the repo's `AGENTS.md`, `.agents/AGENTS.md`, `.factory/AGENTS.md`, or README for project key and Jira instance
2. If not found, ask the user: "Which Jira project key should I use? (e.g., PROJ)"
3. Use `atlassian___getVisibleJiraProjects` to verify the project exists
4. Use `atlassian___getJiraProjectIssueTypesMetadata` to discover available issue types

## Workflow Overview

### Phase 1: Gather Requirements

1. **Determine issue type** - Ask if not clear:
   > "What type of issue are you creating?"
   > - Epic (large initiative containing multiple stories)
   > - Story (new feature/capability for any user, including developers)
   > - Bug (defect/regression)
   > - Task (administrative/maintenance work with no new capability)

   **Issue Type Decision Guide:**
   
   | Type | Definition | Key Question | Examples |
   |------|------------|--------------|----------|
   | **Epic** | Large initiative with business objective, contains multiple stories | "Is this too big for one PR?" | "Multi-tenant support", "New integration" |
   | **Story** | Delivers value or capability to ANY user (end-users OR developers) | "Does this improve someone's workflow or add capability?" | "Add dark mode", "Migrate CI", "Add health endpoint" |
   | **Bug** | Something is broken or not working as expected | "Was this working before?" | "Login fails on Safari", "API returns 500" |
   | **Task** | Administrative/maintenance work, no new capability delivered | "Is this just housekeeping?" | "Rotate API keys", "Update dependencies" |

   **Decision Heuristic:**
   > If it has acceptance criteria that deliver a capability or improve someone's workflow, it's a **Story**, not a Task.
   > "User" includes developers, QA, DevOps - not just end-users of the application.

   **CHECKPOINT:** Before proceeding, explicitly state:
   > "I'm creating this as a [Type] because [reason]."
   
   Wait for user confirmation of the type before gathering details.

2. **Gather context** based on type:

   **For Stories:**
   - What is the user need/problem being solved?
   - Who is the user persona?
   - What are the acceptance criteria?
   - Any designs/mockups?
   - Parent epic (if applicable)?

   **For Bugs:**
   - What is the expected behavior?
   - What is the actual behavior?
   - Steps to reproduce
   - Environment (dev/uat/prod, browser, etc.)
   - Severity/impact
   - Screenshots/logs if available

   **For Tasks:**
   - What administrative/maintenance work needs to be done?
   - Why is it needed? (should NOT be "to improve workflow" - that's a Story)
   - Any dependencies?

   **For Epics:**
   - What is the business objective?
   - What is the scope/boundary?
   - Key milestones or phases?

3. **Identify relationships:**
   - Parent epic to link to?
   - Related issues to link?
   - Blocked by / blocks relationships?

### Phase 2: Structure the Issue

**Check for project-specific templates first:**
- Search for a template epic or wiki page if the user mentions one
- Otherwise use these sensible defaults:

**Story Template:**
```markdown
**Use Case:**
As a [role], I want to [action] so that [benefit].

**Description:**
[Concise explanation of what changes are being made.]

**Acceptance Criteria:**
- [ ] [Specific, measurable criterion]
- [ ] [Specific, measurable criterion]

**Implementation Details:**
[Optional: suggestions for the developer regarding approach.]

**Dependencies Identified:**
[Any dependencies on other stories, teams, or external factors.]
```

**Bug Template:**
```markdown
**Environment:** [Dev/Stage/UAT/Production]

**Description:**
[What is happening]

**Steps to reproduce:**
1. [Step 1]
2. [Step 2]

**Expected behavior:**
[What should happen]

**Actual behavior:**
[What happens instead]
```

**Task Template:**
```markdown
**Objective:**
[What needs to be accomplished]

**Description:**
[Detailed explanation of the work needed]

**Acceptance Criteria:**
- [ ] [Completion criterion 1]
- [ ] [Completion criterion 2]

**Dependencies Identified:**
[Any dependencies on other work]
```

**Epic Template:**
```markdown
**Business Case:**
[Business need or problem this epic addresses, including expected value.]

**Success Criteria:**
[Criteria to determine if the epic is successful.]

**Dependencies:**
[Dependencies on other work or projects.]

**Risks:**
[Potential risks associated with this epic.]

**Stakeholders:**
[Key stakeholders, their roles and responsibilities.]

**Timeline:**
[Estimated timeline with key milestones.]
```

### Phase 3: Preview and Confirm

1. **Show the structured issue:**
   ```
   === JIRA ISSUE PREVIEW ===
   
   Project: [PROJECT_KEY]
   Type: [Story/Bug/Task/Epic]
   Summary: [Title]
   
   Description:
   [Formatted description using template]
   
   Labels: [suggested labels]
   Parent Epic: [if applicable]
   Links: [related issues]
   
   ===========================
   ```

2. **STOP AND ASK:**
   > "Here's the issue I'll create. Would you like to make any changes, or should I create it?"
   
   **Wait for explicit approval before creating.**

### Phase 4: Create the Issue

1. **Create via Atlassian MCP:**
   Use `atlassian___createJiraIssue` with:
   - `projectKey`: [discovered project key]
   - `issueTypeName`: [Story|Bug|Task|Epic]
   - `summary`: [Title]
   - `description`: [Formatted description]

2. **Add links if needed:**
   Use `atlassian___createIssueLink` for:
   - Parent epic relationship
   - Related issue links
   - Blocks/blocked-by relationships

3. **Report success:**
   ```
   Created: [PROJECT]-XXXX
   URL: https://[instance].atlassian.net/browse/[PROJECT]-XXXX
   
   [Summary of what was created and any links added]
   ```

### Phase 5: Follow-up (Optional)

Offer to:
- Create related sub-tasks
- Add to a sprint
- Assign to someone
- Add watchers

## Key Principles

- **Always preview before creating** - User must approve the final structure
- **Use project templates if they exist** - Check for template epics or wiki conventions
- **Include acceptance criteria** - Every story needs testable ACs
- **Link relationships** - Connect to epics and related issues
- **Be specific** - Vague issues lead to implementation confusion

## Common Labels

Suggest labels based on the content:
- `frontend` - UI changes
- `backend` - API/service changes
- `infra` - Infrastructure
- `security` - Security-related
- `performance` - Performance improvements
- `tech-debt` - Technical debt reduction
- `documentation` - Docs updates
