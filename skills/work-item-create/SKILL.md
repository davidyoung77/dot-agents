---
name: work-item-create
description: |
  Create or draft a work item in the active planning system. Use when the user wants
  a new story, bug, task, spike, or work order in 8090, Jira, another tracker, or
  markdown, especially when the right system of record or item type still needs to
  be confirmed.
---

# Work Item Creation

Create planning artifacts without letting the planning tool replace `pdlc`.

## Role In The Workflow

- This skill is the planning-phase adapter inside `pdlc`.
- Use it after requirements are clear enough to produce a scoped work item.
- If the user still needs help defining the problem, scope, acceptance criteria, or architecture, stay in `pdlc` first.
- If the project clearly uses `8090` as the main planning system, default there unless the user explicitly asks for Jira or another tool.
- Keep a single canonical item definition, then map it into the selected tool.

## Step 1: Resolve System Of Record

1. Determine where the item should live: `8090`, Jira, another tracker, both, or draft-only markdown.
2. If the user explicitly names a tool, honor it.
3. If the user does not specify:
   - prefer `8090` when the project already uses `8090` requirements, blueprints, or work orders
   - otherwise ask which tool is authoritative
4. If the user wants dual tracking, ask which system is primary and treat the other as a mirror or backlink target.

## Step 2: Check PDLC Readiness

Before creating anything, decide whether the work is:
- understood work: scope and acceptance criteria are ready
- exploratory work: important unknowns remain

If the work is under-specified:
- do not invent detailed implementation scope
- either route back to `pdlc` or create an explicit spike/discovery item
- call out the missing inputs, such as:
  - problem statement
  - acceptance criteria
  - dependencies
  - requirement or blueprint references
  - parent epic/initiative or phase

## Step 3: Normalize The Work Item

Capture a tool-agnostic item shape:
- title
- item type
- why / problem statement
- scope
- acceptance criteria
- dependencies / blockers
- linked requirements, blueprints, specs, or related items
- verification notes or expected evidence
- primary planning system and any mirror/backlink target

### Generic Type Guide

Use the closest canonical type first, then map it per tool:
- initiative / epic: large multi-item effort
- story / build: new capability
- bug / fix: incorrect behavior or regression
- task / chore: bounded maintenance or coordination work
- spike / discovery: time-boxed research to reduce uncertainty

If the target tool has narrower enums, adapt at creation time instead of changing the meaning of the item.

## Step 4: Gather Tool-Specific Context

Collect only the fields needed for the selected tool.

### 8090

- resolve the project before looking up or creating work orders
- gather phase, parent work order, linked blueprint IDs, and linked requirements when available
- preserve traceability by linking requirements and blueprints instead of burying them in prose
- use the active 8090 planner MCP's `create_work_order` tool when creating the record
- default new items to `backlog` or `ready`; do not mark them `in_progress` until implementation begins

### Jira

- resolve project key and available issue types
- gather parent epic, labels, and issue links
- check for project-specific templates if the user mentions them
- use Atlassian creation and link tools when available
- if Atlassian tools are unavailable, produce a ready-to-paste Jira draft instead of blocking

### Draft-only / Markdown

- return the canonical item as markdown
- include notes on how it would map to `8090` or Jira if the user wants to create it later

### Mapping Guidance

Use these defaults unless the project says otherwise:
- story / build -> `8090` `build`, Jira `Story`
- bug / fix -> `8090` `fix`, Jira `Bug`
- task / chore -> `8090` `other`, Jira `Task`
- spike / discovery -> `8090` `other`, Jira `Task` or project-specific spike type
- requirements drafting -> `8090` `requirements`
- blueprint authoring -> `8090` `blueprint`

## Step 5: Structure The Item

Prepare a clean preview before creating anything:

```markdown
## Work Item Preview

- Primary system: [8090 | Jira | draft]
- Mirror system: [none | Jira | 8090 | other]
- Type: [canonical type]
- Title: [title]

### Why
[problem statement]

### Scope
[what is in and out]

### Acceptance Criteria
- [ ] [criterion]
- [ ] [criterion]

### Dependencies / References
- [requirements / blueprints / blockers / related items]

### Verification Notes
[expected tests, screenshots, API evidence, or reviewer focus]
```

Always show the preview and ask for approval before creating anything.

## Step 6: Create The Record

After approval, create the item in the chosen system.

### 8090 Creation Flow

1. Resolve the project and, when needed, linked blueprint or requirement IDs.
2. Use `create_work_order` with:
   - `title`
   - `description_markdown`
   - `type`
   - `status` only if the user explicitly wants something other than the default
   - optional `phase_number`, `parent_work_order_id`, `blueprint_ids`, or assignee
3. Report the created work order number and any linked artifacts.

### Jira Creation Flow

1. Resolve project key and target issue type.
2. Use `createJiraIssue` with the approved title and description.
3. Add parent epic or related links after creation when needed.
4. Report the created issue key and any links added.

### Dual-Tracking Flow

1. Create the primary record first.
2. Add the primary item reference into the secondary record or comment when the tool supports it.
3. If backlink automation is unavailable, return the exact text the user can paste later.

## Step 7: Follow-Up

Offer the next appropriate step:
- create another related work item
- link this item to requirements, blueprints, or epics
- switch to `work-item-implement`
- move back into `pdlc` if the item exposed missing requirements or architecture work

## Principles

- `pdlc` owns lifecycle discipline; this skill only creates planning artifacts
- do not let Jira or any tracker become the implicit source of truth by accident
- default to `8090` when the project already uses `8090`, but stay explicit about the chosen system of record
- preview before create
- preserve traceability: requirement -> blueprint/spec -> work item
- if tool access is unavailable, produce the best draft instead of hallucinating success
