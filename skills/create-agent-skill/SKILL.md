---
name: create-agent-skill
description: |
  Create a new skill following the ~/.agents/ shared pattern.
  Use when the user says "create skill", "new skill", or wants to add a skill
  that works across Factory and Cursor.
---

# Create Agent Skill

Guide the user through creating a skill in the shared `~/.agents/` structure.

## Gather Requirements

1. **Purpose**: What should this skill do?
2. **Scope**: Personal (`~/.agents/skills/`) or project-specific (`.agents/skills/`)?
3. **Trigger**: When should the agent apply this skill?

If the user has previous conversation context, infer answers from what was discussed.

## Create the Skill

### Personal skill (available across all projects)

```
~/.agents/skills/<skill-name>/SKILL.md
```

After creating, run `~/.agents/bin/sync-to-tools` to symlink it into Factory and Cursor.
Add the skill name to the `ALL_SKILLS` array in `~/.agents/bin/sync-to-tools`.

### Project-specific skill (shared with the repo)

```
.agents/skills/<skill-name>/SKILL.md
```

Then symlink into each tool's project config:
```bash
mkdir -p .factory/skills/<skill-name> .cursor/skills/<skill-name>
ln -s "$(pwd)/.agents/skills/<skill-name>/SKILL.md" .factory/skills/<skill-name>/SKILL.md
ln -s "$(pwd)/.agents/skills/<skill-name>/SKILL.md" .cursor/skills/<skill-name>/SKILL.md
```

## SKILL.md Format

```markdown
---
name: skill-name
description: |
  Brief description of what this skill does and when to use it.
---

# Skill Title

## Steps
Clear, step-by-step guidance for the agent.
```

### Frontmatter Rules
- `name`: lowercase, hyphens, max 64 chars
- `description`: max 1024 chars, include WHAT it does and WHEN to use it

## Best Practices

- Keep SKILL.md under 500 lines
- Use progressive disclosure: put details in separate reference files
- Provide concrete examples, not abstract instructions
- If the skill needs shared instruction files, put them in `~/.agents/instructions/`
  and reference them from the skill
- Use one term consistently throughout (don't mix synonyms)

## After Creation

1. Verify the SKILL.md is well-formed
2. For personal skills: run `~/.agents/bin/sync-to-tools` to symlink
3. For project skills: create symlinks manually (shown above)
4. Test the skill by invoking it in a conversation
