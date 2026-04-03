# dot-agents

Portable, tool-agnostic AI agent configuration. Single source of truth for
personal sub-agents, skills, rules, and coding standards — synced to
[Factory](https://factory.ai) and [Cursor](https://cursor.com) via a mix of
symlinks and generated/copied files.

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
│   ├── code-reviewer-opus.md    # Reviewer family for review diversity
│   ├── code-reviewer-gpt.md
│   ├── code-reviewer-gemini.md
│   ├── spec-reviewer-*.md       # Spec reviewers (3 variants)
│   ├── prd-reviewer-*.md        # PRD reviewers (3 variants)
│   └── shared/                  # Only for instruction bodies reused by 2+ agents
│       ├── code-review.md
│       ├── spec-review.md
│       └── prd-review.md
├── rules/                       # Shared rules applied to all conversations
│   └── diversity-review.md      # Capability-aware review diversity guidance
└── skills/                      # Reusable workflow skills (14)
    ├── commit/                  # Conventional commit workflow
    ├── pr-create/               # Pull request creation
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

**Skills** are symlinked into both tools. **Sub-agents** are symlinked into
Factory and copied as real files into `~/.cursor/agents/` because Cursor
subagent discovery is not reliable with symlinks. **Rules** are symlinked to
Factory and generated as `.mdc` files for Cursor (which requires inline YAML
frontmatter).

For Cursor specifically, treat `~/.cursor/agents/` as the documented global
path, but not the only reliable runtime path. If Cursor does not surface a
personal sub-agent in a given repo, copy the needed agent into that repo's
`.cursor/agents/` directory as a real file.

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

- **Source of truth over tool-local edits**: `~/.agents/` remains canonical.
  Skills are symlinked to both tools, Factory droids are symlinked, and Cursor
  user-scope sub-agents are copied as real files because symlink discovery is
  unreliable there.
- **Unified frontmatter**: One file serves both Factory (droids) and Cursor
  (agents). Each tool reads its own fields and ignores the rest.
- **Memory at orchestrator level**: Sub-agents don't manage memory. The
  orchestrator (main agent) reads `memories.md` via the always-on AGENTS.md rule.
- **Repo-local runtime artifacts**: For project repos, ephemeral agent output
  (reports, payloads, working files) belongs under `.agents/work/` and the whole
  tree should be gitignored.
- **Diversity-of-opinion reviews**: Prefer true cross-model review diversity on
  platforms that support distinct model assignment per reviewer; otherwise use
  prompt-diverse reviewer families and be explicit about the limitation.
