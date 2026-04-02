---
name: create-agent-skill
description: |
  Create a new skill, sub-agent, command, or rule following the ~/.agents/ shared pattern.
  Use when the user says "create skill", "new skill", "new agent", "new droid",
  "new command", or wants to add an agent artifact that works across Factory and Cursor.
---

# Create Agent Artifact

Guide the user through creating agent artifacts in the `~/.agents/` structure.
This follows the [Agent Skills community standard](https://agentskills.io/specification)
([GitHub](https://github.com/agentskills/agentskills)) — `.agents/skills/` is the
canonical location recognized by 40+ AI coding tools.

## Gather Requirements

1. **Type**: Skill, sub-agent, command, or rule?
2. **Scope**: Personal (`~/.agents/`) or project-specific (`.agents/`)?
3. **Purpose**: What should it do?
4. **Trigger**: When should the agent apply this?
5. **Key knowledge**: What does the agent need that it wouldn't already know?

If you have previous conversation context, infer answers from what was discussed.

## Artifact Types

### Skills

Interactive guides that walk the agent through a multi-step workflow.

All skills live in `.agents/skills/` — the community-standard path.
Run `sync-to-tools` after adding a new top-level skill directory so both
Cursor and Factory pick it up.

```bash
mkdir -p ~/.agents/skills/<skill-name>
```

Write `~/.agents/skills/<skill-name>/SKILL.md` following the
[Agent Skills spec](https://agentskills.io/specification):

```markdown
---
name: skill-name
description: |
  Brief description of WHAT this skill does and WHEN to use it.
  Include trigger terms the agent can match on.
---

# Skill Title

## Steps
Clear, step-by-step guidance for the agent.
```

#### Frontmatter rules

- `name`: lowercase, hyphens only, max 64 chars, must match directory name
- `description`: max 1024 chars, third person, include WHAT and WHEN
- See the full [spec](https://agentskills.io/specification) for optional fields
  (`license`, `compatibility`, `metadata`, `allowed-tools`)

#### Supporting files

For detailed reference material, use progressive disclosure:

```
~/.agents/skills/<skill-name>/
├── SKILL.md              # Required — main instructions (under 500 lines)
├── references/           # Optional — detailed docs loaded on demand
├── scripts/              # Optional — executable code
└── assets/               # Optional — templates, resources
```

Keep SKILL.md concise. Link to reference files for details:
`For API details, see [references/api.md](references/api.md)`

### Sub-agents (Cursor agents / Factory droids)

Specialized agents spawned by commands or other agents to handle focused tasks.

All sub-agents live in `~/.agents/sub-agents/` (source of truth).
The sync script symlinks each `.md` file to `.cursor/agents/` and `.factory/droids/`.

```bash
mkdir -p ~/.agents/sub-agents
```

Write `~/.agents/sub-agents/<agent-name>.md` (flat file, not a directory):

```markdown
---
name: agent-name
description: Brief description of what this sub-agent does.
model: fast
---

# Agent Title

You are a specialized agent. [system prompt instructions here]
```

Sub-agents are flat `.md` files — no `SKILL.md` nesting.
Use a `shared/` subdirectory for reference files multiple sub-agents share.

### Commands

Custom slash commands (`/command-name`) shared between Cursor and Factory.

All commands live in `~/.agents/commands/` (source of truth).
The sync script creates directory-level symlinks to `.cursor/commands/`
and `.factory/commands/`.

```bash
mkdir -p ~/.agents/commands
# Write a command as markdown (filename becomes /command-name)
touch ~/.agents/commands/<command-name>.md
```

Commands support optional YAML frontmatter (`description`, `argument-hint`)
and `$ARGUMENTS` expansion.

### Rules

Coding standards and conventions the agent should always follow.

Rules live in `~/.agents/rules/` as `.md` files. The sync script:
- Symlinks to `.factory/rules/`
- Generates Cursor `.mdc` wrappers with appropriate frontmatter

```bash
echo "# My Rule" > ~/.agents/rules/<rule-name>.md
```

## After Creation

Run `~/.agents/bin/sync-to-tools` to activate the new artifact.
Skills and commands use directory-level symlinks, so new files within
already-synced directories are available immediately without re-running.

For project-specific artifacts, follow the repo's own setup/sync instructions
if present (for example in `AGENTS.md`, `.agents/AGENTS.md`, repo scripts, or
git hooks).

## Best Practices

- Keep SKILL.md under 500 lines — the context window is shared with conversation
- Only add knowledge the agent doesn't already have
- Use consistent terminology (pick one term, stick with it)
- Provide concrete examples, not abstract instructions
- For fragile operations, use scripts over text instructions
- Ensure artifacts are tool-neutral — avoid APIs specific to one tool
