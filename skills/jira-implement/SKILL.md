---
name: jira-implement
description: |
  Jira adapter over `work-item-implement`. Use when the user explicitly wants to
  implement work from a Jira issue or when Jira remains the visible task reference.
---

# Jira Work Item Implementation

This is a thin Jira adapter over `work-item-implement`.

## When To Use

- The user points to a Jira key such as `PROJ-123`
- Jira is the visible tracker for the requested work
- The project wants Jira comments or transitions during implementation

If the project primarily uses `8090` or another tool, prefer `work-item-implement` and use Jira only as context or a mirror.

## Follow The Canonical Workflow

Follow `work-item-implement` as the source of truth, with these Jira-specific overrides.

### Jira Overrides

1. Read the Jira issue and linked context first.
2. Treat the Jira issue as planning context, not automatic technical authority.
3. If the issue links to a primary `8090` item, blueprint, PRD, or repo spec, read that before planning code changes.
4. If the Jira issue is under-specified for anything beyond a small bug fix, stop and route back to `pdlc` or create a discovery item.
5. Use Jira transitions or comments only when the user wants tracker updates.
6. After PR creation, offer a Jira comment with the PR link and short summary.

### Jira-Specific Checks

- confirm whether the ticket assumptions still match the codebase
- check for recent merged PRs or code changes that may stale the ticket
- flag scope drift before implementing

## Principles

- `work-item-implement` owns the implementation workflow
- Jira is an adapter, not the technical source of truth
- keep traceability back to requirements, blueprints, and primary work items
- do not auto-transition to a review-ready state without user approval
