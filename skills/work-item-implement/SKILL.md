---
name: work-item-implement
description: |
  Implement a single work item from the active planning system. Use when the user
  wants to execute a story, bug, task, or work order from 8090, Jira, another
  tracker, or a markdown brief, while keeping PDLC traceability intact.
---

# Work Item Implementation

Implement a single work item as the implementation subflow inside `pdlc`.

## Role In The Workflow

- Use this skill for one scoped item or a small, loosely related set.
- If 4+ interdependent items need to move together, switch to `mission`.
- Read the work item, then read the linked requirements, blueprints, or specs before writing code.
- If the project clearly uses `8090` as the main planning system, default there unless the user explicitly points to Jira or another tool.
- Defer commit and PR publication to `commit` and `pr-create`.

## Step 1: Resolve The Work Item

1. Determine the source system: `8090`, Jira, another tracker, or markdown.
2. Fetch the item details and normalize:
   - ID / key
   - title
   - scope
   - acceptance criteria
   - dependencies
   - linked requirements, blueprints, specs, or related items
3. If the user only provides a secondary mirror item, ask whether there is a primary spec or work item elsewhere.

## Step 2: Enforce The Traceability Gate

Before planning code changes, verify the implementation context is strong enough.

### Minimum Context

- work item scope and acceptance criteria
- technical references when complexity warrants them
- known dependencies or blockers

### Required Behavior

- For `8090`, read linked blueprints and requirements whenever they exist.
- For Jira, treat the Jira issue as planning context, not automatic technical authority.
- If the item is medium/high complexity and there is no linked spec or blueprint, stop and route back to `pdlc` for clarification or create a discovery item.
- For small, self-evident bug fixes, proceed only when the behavior and acceptance criteria are clear.

## Step 3: Right-Size The Delivery Method

- One isolated item: stay in this skill
- Small related set with little file overlap: sequential sessions are fine
- 4+ interdependent items or a shared-file batch: switch to `mission`

Call the escalation out explicitly instead of stretching this skill into mission behavior.

## Step 4: Validate Current-State Assumptions

Before implementation:
- inspect the affected code paths
- check for recent changes that may stale the item's assumptions
- flag mismatches between the item and current code to the user
- recommend one of:
  - proceed as written
  - adjust scope
  - clarify requirements or blueprint first

## Step 5: Plan And Ask For Approval

Create a focused plan with:
- files to modify
- main code changes
- tests to add or update
- runtime validation if warranted
- risks, dependencies, and any blueprint or requirements drift to watch

Show the plan and wait for explicit approval before changing code.

## Step 6: Prepare To Implement

After approval:
1. Follow repo conventions from `AGENTS.md` and nearby code.
2. Use the existing branch if it fits the work; otherwise create a descriptive branch that matches project conventions.
3. Update work item status when actual implementation begins:
   - `8090`: set `in_progress`
   - Jira: move to the appropriate in-progress transition if the user wants tracker status maintained during execution
4. Do not update to a review-ready state without user approval.

## Step 7: Implement And Verify

1. Make the code changes.
2. Add or update automated tests when they materially reduce regression risk.
3. Run the repo's normal validation:
   - lint
   - typecheck
   - build when relevant
   - automated tests
4. Decide whether runtime validation is warranted for changed UI, APIs, auth, integrations, or stateful flows.
5. If runtime validation needs services, browser automation, or credentials, show a short validation plan and ask before running it.
6. Capture evidence clearly for the later PR:
   - commands run
   - what passed, failed, or was skipped
   - screenshots or API evidence when relevant

## Step 8: Review And Handoff

Before finishing:
- surface requirement or blueprint drift if you found any
- use reviewer passes when the change is significant or ambiguous, especially before a PR
- hand off to `commit` for the commit
- hand off to `pr-create` for the PR

## Step 9: Update The Work Item Carefully

After implementation and tests:
- `8090`: ask before setting `in_review`; do not do it automatically
- Jira: offer a Jira comment or transition only if the user wants it
- if there is a mirrored secondary tracker, update or draft the backlink text after the primary system is updated

## Tool Notes

### 8090

- Use the active 8090 planner MCP to read the work order plus any linked blueprints and requirements.
- `8090` is strongest when the traceability chain stays intact.
- Prefer linked blueprint and requirement references over copying large specs into chat.

### Jira

- Use Atlassian tools to read the ticket and transitions when available.
- Jira issues can point at the work, but linked specs, blueprints, or repo docs are still the technical source of truth.
- If Jira is only a mirror for a primary `8090` item, resolve the `8090` context before implementation.

### Markdown / Ad Hoc Briefs

- Normalize the brief into the same fields as a tracked work item.
- If the work becomes recurrent or involves coordination, recommend creating a real tracked item before implementation.

## Principles

- read the work item, then the blueprint/spec, then the code
- do not let a stale ticket drive implementation without validation
- `mission` handles multi-item batches; this skill stays single-item
- `commit` and `pr-create` own publication steps
- do not move tracker status to a review-ready state without the user's approval
