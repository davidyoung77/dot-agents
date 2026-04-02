---
name: sync-to-tools
description: |
  Sync personal ~/.agents/ config to Factory and Cursor (skills, sub-agents, rules, MCP).
  Use when the user says "sync", "sync to tools", "update tools", or wants to
  push config changes to their tool setup.
---

# Sync to Tools

Run the sync script to update Factory and Cursor from the shared agent config.

## Steps

1. Run `~/.agents/bin/sync-to-tools`
2. Report the summary line to the user
3. If warnings appear, explain what manual action is needed

## Dry Run

If the user wants to preview first, add `--dry-run`:

```bash
~/.agents/bin/sync-to-tools --dry-run
```

## What Syncs

All items are auto-discovered — no hardcoded lists in the script.

| Source | Targets | Method |
|--------|---------|--------|
| `~/.agents/mcp.json` | `.factory/mcp.json`, `.cursor/mcp.json` | symlink |
| `~/.agents/skills/*/` | `.factory/skills/<name>`, `.cursor/skills-cursor/<name>` | symlink |
| `~/.agents/sub-agents/*.md` | `.factory/droids/`, `.cursor/agents/` | symlink |
| `~/.agents/commands/` | `.factory/commands/`, `.cursor/commands/` | symlink |
| `~/.agents/AGENTS.md` | `.factory/rules/coding-standards.md` | symlink |
| `~/.agents/AGENTS.md` | `.cursor/rules/personal-coding-standards.mdc` | generated .mdc |
| `~/.agents/rules/*.md` | `.factory/rules/`, `.cursor/rules/*.mdc` | symlink + generated .mdc |

## Direction

`~/.agents/` is the source of truth. Edit sub-agents, skills, rules, and AGENTS.md
there, then run this skill to push changes to both tools. Adding a new top-level
entry under `sub-agents/`, `skills/`, `rules/`, or `commands/` is picked up on the
next sync — no script edits needed.

## Script Location

`~/.agents/bin/sync-to-tools`
