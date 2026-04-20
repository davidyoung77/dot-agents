---
name: jira-create
description: |
  Jira adapter over `work-item-create`. Use when the user explicitly wants a Jira
  issue created or drafted from a story, bug, task, spike, or initiative.
---

# Jira Issue Creation

This is a thin Jira adapter over `work-item-create`.

## When To Use

- The user explicitly wants a Jira issue
- The project still uses Jira as the primary tracker
- The user wants a mirrored Jira record alongside a primary item elsewhere

If the user only needs a generic planning artifact or the project primarily uses `8090`, prefer `work-item-create` and select Jira only as the output target.

## Follow The Canonical Workflow

Follow `work-item-create` as the source of truth, with these Jira-specific overrides.

### Jira Overrides

1. Resolve Jira project key and issue types.
2. Map the canonical item type into the best Jira issue type.
3. Gather Jira-only fields:
   - parent epic
   - labels
   - related issue links
   - project-specific template needs
4. Always preview before create.
5. Use Atlassian creation and link tools when available.
6. If Atlassian access is unavailable, return a ready-to-paste Jira draft instead of blocking.

### Mirror Behavior

- If `8090` or another tracker is the primary system, create or resolve that item first when the user wants cross-tool traceability.
- Add the primary item reference to the Jira description or a follow-up comment when the workflow supports it.
- Do not let the mirrored Jira issue silently become the only source of truth.

### Jira Draft Shape

```markdown
Project: [KEY]
Type: [Story/Bug/Task/Epic]
Summary: [title]

Description:
[approved canonical description]

Links / Epic / Labels:
- ...
```

## Principles

- `work-item-create` owns the canonical workflow
- Jira is an adapter, not the lifecycle model
- preserve links back to requirements, blueprints, and the primary work item when they exist
- preview before create
