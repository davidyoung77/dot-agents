---
name: mission
description: |
  Prepare mission briefs and operator runbooks for bulk implementation of 4+ interdependent
  work items from any source (8090, Jira, Linear, markdown list). Tool-agnostic at the
  core model level, with concrete output modes such as Factory mission prompts, generic
  markdown runbooks, and local orchestrator briefs.
---

# Mission Skill

You help the user prepare and execute mission-style bulk implementation workflows for interdependent work items.

## Role In The Workflow

- Use this skill after requirements, architecture, and planning have already produced scoped work items with acceptance criteria and dependencies.
- This skill is the bulk-implementation subflow inside `pdlc`; it does not replace the broader lifecycle.
- Work items may come from 8090, Jira, Linear, or manual lists.
- The core mission model is tool-agnostic: milestones, dependencies, status conventions, test commands, off-limits, and post-mission validation steps.
- Ask which target output mode the user wants before generating the final artifact: `factory`, `generic_markdown`, or `local_orchestrator`.
- If the user already implies a target tool, use it. If not, default to `generic_markdown`.
- This skill helps the orchestrator prepare the mission brief, milestone structure, PM status instructions, and post-mission validation steps.
- This skill is not a durable mission runtime: pause/resume, supervision, persistent state, and recovery remain the responsibility of the orchestrator and tooling around it.

## When to Use Missions

Missions are appropriate when:
- 4+ interdependent work items need implementation together
- Items share files, schemas, or architectural patterns
- Sequential implementation would cause constant rework
- A phase or epic forms a natural batch

Missions are NOT appropriate when:
- Single tickets or isolated bug fixes
- Items have no shared dependencies
- Work is exploratory or research-only; stay in `pdlc` and run discovery/spike work first

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

## Step 3: Choose Output Mode

Before rendering the final artifact, confirm the target output mode:

- `factory`: Generate a Factory-style mission prompt and operator runbook.
- `generic_markdown`: Generate a tool-neutral mission brief in markdown.
- `local_orchestrator`: Generate a brief optimized for the main session to orchestrate workers directly.

## Step 4: Generate Canonical Mission Brief

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
- If the PM tool does not have an explicit review-ready state, adapt this to the closest equivalent

[Include the specific tool call syntax for the user's PM tool, e.g.:]
[8090: edit_work_order(work_order_id=UUID, status="in_progress")]
[Jira: atlassian___transitionJiraIssue(issueIdOrKey, transition)]

## Conventions
- Follow the project's shared coding standards (`AGENTS.md`, `.agents/AGENTS.md`, `.factory/AGENTS.md`, or equivalent)
- Git commits: `feat(ITEM-ID): description`
- Typecheck and unit tests must pass at each milestone boundary

## Test Commands
- [PROJECT-SPECIFIC: e.g., npx vitest run, npx nx run-many --target=typecheck --all]
- [Include all relevant test/lint commands for the project]

## Off-Limits
- [Any packages/files the mission should not modify]
```

### Render By Output Mode

**Factory**
- Render the canonical brief as a Factory-style mission prompt.
- Keep PM status instructions inline with the prompt.
- Preserve the post-mission checklist for the orchestrator.

**Generic Markdown**
- Render the canonical brief as plain markdown with no Factory-specific wording.
- Preserve milestones, dependencies, constraints, status instructions, and the post-mission checklist.

**Local Orchestrator**
- Render the canonical brief as an orchestrator runbook for the main session.
- Call out what stays in the orchestrator, what should be delegated to workers, and which review agents should run after implementation.

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

## Step 5: Post-Mission Checklist

After the mission completes, present this checklist to the orchestrator (the user/Droid session running the mission):

```
## Post-Mission Review

Mission complete. Run these steps before marking items as done:

### 1. Code Review (parallel)
Launch the code reviewer family against the mission's changes.
Prefer true cross-model diversity where the platform supports distinct models per reviewer.
Otherwise, use the available reviewer variants as prompt-diverse passes and note the limitation.

### 2. Spec Review (parallel, if blueprints exist)
Launch the spec reviewer family to check for blueprint/spec drift, using true model diversity when supported.

### 3. PRD/Requirements Review (parallel, if requirements docs exist)
Launch the PRD reviewer family for requirements coverage, using true model diversity when supported.

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
- Reviewer agents/droids live in `~/.agents/sub-agents/` and are synced into the supported tools.
- If some reviewer passes timeout or fail, results from the remaining useful passes are sufficient when there is still enough signal.
- Smoke tests catch what reviewers can't: shared resource contention, connection pool exhaustion, cold-start hangs.
