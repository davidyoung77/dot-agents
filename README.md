# dot-agents

Portable, tool-agnostic AI agent configuration. Single source of truth for
personal sub-agents, skills, rules, and coding standards — synced to
[Factory](https://factory.ai) and [Cursor](https://cursor.com) via symlinks.

## Structure

```
~/.agents/
├── AGENTS.md                    # Portable coding standards + memory reference
├── mcp.json                     # MCP server config (gitignored, see .example)
├── memories.md                  # Personal decision log (gitignored)
├── bin/
│   └── sync-to-tools            # Idempotent sync script
├── sub-agents/                  # Unified agent definitions
│   ├── builder.md               # Implementation agent
│   ├── code-reviewer-opus.md    # 3-model diversity pattern
│   ├── code-reviewer-gpt.md
│   ├── code-reviewer-gemini.md
│   ├── spec-reviewer-*.md       # Spec reviewers (3 variants)
│   ├── prd-reviewer-*.md        # PRD reviewers (3 variants)
│   └── shared/                  # Only for instruction bodies reused by 2+ agents
│       ├── code-review.md
│       ├── spec-review.md
│       └── prd-review.md
├── rules/                       # Shared rules applied to all conversations
│   └── diversity-review.md      # Fan-out to 3 model variants for reviews
└── skills/                      # Reusable workflow skills (14)
    ├── commit/                  # Conventional commit workflow
    ├── pr/                      # Pull request creation
    ├── pr-review/               # PR review orchestration
    ├── pdlc/                    # Product development lifecycle
    └── ...
```

## How It Works

**Sub-agents** use unified YAML frontmatter that both tools parse, each reading
only the fields it understands. Keep instructions inline by default; use `shared/`
only when multiple agents reuse the same instruction body:

```yaml
---
name: code-reviewer-opus
description: Code reviewer (Opus 4.6)
model: claude-opus-4-6        # Both tools
reasoningEffort: max           # Factory reads, Cursor ignores
tools: ["Read", "Grep", "Glob"] # Factory reads, Cursor ignores
readonly: true                 # Cursor reads, Factory ignores
---
Before doing anything else, read the shared instructions file at
`~/.agents/sub-agents/shared/code-review.md` and follow them exactly.
```

**Skills** and **sub-agents** are symlinked into both tools. **Rules** are
symlinked to Factory and generated as `.mdc` files for Cursor (which requires
inline YAML frontmatter).

## Setup

1. Clone to `~/.agents/`:
   ```bash
   git clone git@github.com:davidyoung77/dot-agents.git ~/.agents
   # or via HTTPS:
   git clone https://github.com/davidyoung77/dot-agents.git ~/.agents
   ```

2. Copy `mcp.json.example` to `mcp.json` and add your API keys:
   ```bash
   cp ~/.agents/mcp.json.example ~/.agents/mcp.json
   ```

3. Run the sync script:
   ```bash
   ~/.agents/bin/sync-to-tools
   ```

   This creates symlinks in `~/.factory/` and `~/.cursor/`. Safe to re-run
   anytime — it's idempotent and auto-discovers new skills, sub-agents, and
   rules without script edits.

4. Preview changes without writing:
   ```bash
   ~/.agents/bin/sync-to-tools --dry-run
   ```

## Adding New Assets

**New skill**: Create `skills/<name>/SKILL.md`, run `sync-to-tools`.

**New sub-agent**: Create `sub-agents/<name>.md` with unified frontmatter,
run `sync-to-tools`. Or use the `create-sub-agent` skill.

**New rule**: Create `rules/<name>.md`, run `sync-to-tools`.

No script edits needed — everything is auto-discovered.

## Design Decisions

- **Symlinks over generation**: Sub-agents and skills are symlinked, not copied.
  Edit once in `~/.agents/`, both tools see the change immediately.
- **Unified frontmatter**: One file serves both Factory (droids) and Cursor
  (agents). Each tool reads its own fields and ignores the rest.
- **Memory at orchestrator level**: Sub-agents don't manage memory. The
  orchestrator (main agent) reads `memories.md` via the always-on AGENTS.md rule.
- **Diversity-of-opinion reviews**: 3 model variants per review type, with a
  shared rule that tells the orchestrator to fan out and synthesize results.
