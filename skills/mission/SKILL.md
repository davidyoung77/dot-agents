---
name: mission
description: |
  Generate structured Factory Mission prompts from work items. Use when planning a mission
  with 4+ interdependent work items from any source (8090, Jira, Linear, markdown list).
  Handles prompt generation, work item status conventions, and post-mission review checklist.
---

# Factory Mission Prompt Generator

You help the user prepare and execute Factory Missions for bulk implementation of interdependent work items.

## When to Use Missions

Missions are appropriate when:
- 4+ interdependent work items need implementation together
- Items share files, schemas, or architectural patterns
- Sequential implementation would cause constant rework
- A phase or epic forms a natural batch

Missions are NOT appropriate when:
- Single tickets or isolated bug fixes
- Items have no shared dependencies
- Work is exploratory or research-only

## Step 1: Gather Work Items

Collect work items from whatever source the user has. Ask what tool they're using:

- **8090 Planner**: Use `list_work_orders(phase_number, status)` and `read_work_order(work_order_number)`
- **Jira**: Use `atlassian___searchJiraIssuesUsingJql` or `atlassian___getJiraIssue`
- **Linear**: Use Linear MCP tools if available
- **Manual list**: User provides titles, descriptions, acceptance criteria directly

For each item, extract: **ID, title, description/scope, acceptance criteria, dependencies**.

## Step 2: Group into Milestones

Organize work items into 2-5 milestones based on dependencies:
- Items that must exist before others go in earlier milestones
- Items touching the same files/modules group together
- Each milestone should be independently testable

Present the grouping to the user for approval before generating the prompt.

## Step 3: Generate Mission Prompt

Use this template, filling in project-specific details:

```markdown
## Scope
Implement [PHASE/EPIC NAME]: [N] work items across [M] milestones.

## Work Items
### Milestone 1: [NAME] ([ITEM-IDs])
- [ITEM-ID]: [Title] — [1-line scope summary]
- ...

### Milestone 2: [NAME] ([ITEM-IDs])
- ...

## Work Item Status Management
Update each work item's status as you progress:
- **Starting work**: Set status to "in_progress" (via [TOOL_NAME])
- **Implementation + tests pass**: Set status to "in_review" (via [TOOL_NAME])
- Never leave items in backlog/todo once work has begun
- Update per-item, not in bulk at milestone end

[Include the specific tool call syntax for the user's PM tool, e.g.:]
[8090: edit_work_order(work_order_id=UUID, status="in_progress")]
[Jira: atlassian___transitionJiraIssue(issueIdOrKey, transition)]

## Conventions
- Follow `.factory/AGENTS.md` (or project AGENTS.md) for architecture and code style
- Git commits: `feat(ITEM-ID): description`
- Typecheck and unit tests must pass at each milestone boundary

## Test Commands
- [PROJECT-SPECIFIC: e.g., npx vitest run, npx nx run-many --target=typecheck --all]
- [Include all relevant test/lint commands for the project]

## Off-Limits
- [Any packages/files the mission should not modify]
```

### Tool-Specific Status Instructions

**8090 Planner:**
```
Use edit_work_order with work_order_id and status fields.
Statuses: backlog → in_progress → in_review
```

**Jira:**
```
Use getTransitionsForJiraIssue to discover available transitions, then
transitionJiraIssue to move the issue. Typical flow: To Do → In Progress → In Review
```

**No PM tool:**
```
Omit the status management section entirely.
```

## Step 4: Post-Mission Checklist

After the mission completes, present this checklist to the orchestrator (the user/Droid session running the mission):

```
## Post-Mission Review

Mission complete. Run these steps before marking items as done:

### 1. Code Review (parallel)
Launch 3 reviewer droids against the mission's changes:
- code-reviewer-opus
- code-reviewer-gpt  
- code-reviewer-gemini

### 2. Spec Review (parallel, if blueprints exist)
Launch 3 spec reviewer droids to check for blueprint/spec drift:
- spec-reviewer-opus
- spec-reviewer-gpt
- spec-reviewer-gemini

### 3. PRD/Requirements Review (parallel, if requirements docs exist)
Launch 3 PRD reviewer droids for requirements coverage:
- prd-reviewer-opus
- prd-reviewer-gpt
- prd-reviewer-gemini

### 4. Fix Findings
Dispatch parallel workers to fix any critical/warning findings.

### 5. Smoke Test
If the project has a smoke-test skill or integration test suite,
run it against real services (Docker Compose, local dev, etc.).
Unit tests mock infrastructure — integration bugs only surface here.

### 6. Mark Complete
After all reviews pass and smoke test is clean, mark work items
as completed in the PM tool.
```

## Notes

- The mission itself should NOT run reviewers. That's the orchestrator's job post-mission.
- Reviewer droids are personal droids in `~/.factory/droids/` — they work across any project.
- If some reviewer models timeout (common with gemini), results from 2/3 models is sufficient.
- Smoke tests catch what reviewers can't: shared resource contention, connection pool exhaustion, cold-start hangs.
