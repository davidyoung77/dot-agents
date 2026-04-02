---
name: create-sub-agent
description: Create a new sub-agent definition in ~/.agents/ with unified frontmatter for Factory and Cursor
---

# Create Sub-Agent

Use this skill to create a new sub-agent that works in both Factory and Cursor via symlinks.

## Steps

1. **Gather requirements** — ask the user for:
   - Agent name (lowercase, hyphenated, e.g. `security-auditor`)
   - One-line description
   - Model (`inherit`, `fast`, or a specific ID like `claude-opus-4-6`, `gpt-5.4`)
   - Read-only? (`true` for reviewers/analyzers, `false` for builders/implementers)
   - Factory reasoning effort (`low`, `medium`, `high`, `xhigh`, `max`) — skip if model is `inherit`
   - Factory tools list (e.g. `["Read", "Grep", "Glob", "LS"]`) — optional
   - Whether instructions should be inline or shared with other agents

2. **Create the sub-agent definition** at `~/.agents/sub-agents/<name>.md`:

By default, keep instructions inline in the sub-agent file. Use `shared/` only when the same instruction body will be referenced by 2 or more agents.

```markdown
---
name: <name>
description: <description>
model: <model>
reasoningEffort: <effort>
tools: <tools-array>
readonly: <true|false>
---
<inline instructions here>
```

Only include `reasoningEffort` and `tools` if values were provided. Omit them for `model: inherit` agents.

3. **Create shared instructions only when needed** at `~/.agents/sub-agents/shared/<instruction-file>.md`:
   - Write tool-neutral instructions (no Factory or Cursor framing)
   - Focus on the task, checklist, and expected output format
   - Only create a shared file when multiple sub-agents will reference it (e.g. the 3-model reviewer pattern)
   - In that case, the sub-agent file should point to the shared file:

```markdown
---
name: <name>
description: <description>
model: <model>
reasoningEffort: <effort>
tools: <tools-array>
readonly: <true|false>
---
Before doing anything else, read the shared instructions file at `~/.agents/sub-agents/shared/<instruction-file>.md` and follow them exactly.
```

4. **For multi-model variants** (diversity-of-opinion pattern):
   - Create one instruction file in `shared/`
   - Create 2-3 sub-agent definitions with different models, all referencing the same instruction file
   - Name them `<role>-opus.md`, `<role>-gpt.md`, `<role>-gemini.md`

5. **Run sync** — execute `~/.agents/bin/sync-to-tools` to create symlinks in both `.factory/droids/` and `.cursor/agents/`.

6. **Verify** — confirm the symlinks exist and both tools can see the new agent:
   - `ls -la ~/.factory/droids/<name>.md`
   - `ls -la ~/.cursor/agents/<name>.md`

## Unified Frontmatter Reference

| Field | Used by | Required | Notes |
|---|---|---|---|
| `name` | Both | Yes | Lowercase, hyphenated |
| `description` | Both | Yes | Agent reads this to decide delegation |
| `model` | Both | No | `inherit` (default), `fast`, or specific ID |
| `reasoningEffort` | Factory | No | `low`/`medium`/`high`/`xhigh`/`max` |
| `tools` | Factory | No | JSON array of tool names |
| `readonly` | Cursor | No | `true` restricts write tools |
| `is_background` | Cursor | No | `true` for non-blocking execution |
