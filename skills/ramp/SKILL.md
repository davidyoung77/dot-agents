---
name: ramp
description: |
  Systematically onboard onto a new codebase. Reads project config, maps structure,
  discovers conventions, test commands, and project agent assets. Use when opening a new
  project or saying "ramp me up", "explore this codebase", or "what is this project".
---

# Codebase Ramp

Systematically explore and summarize a codebase so you can work effectively in it.

## Steps

### 1. Project Identity

Read these files (skip any that don't exist):
- `README.md` / `README`
- `AGENTS.md` / `.agents/AGENTS.md` / `.factory/AGENTS.md`
- `package.json` / `pyproject.toml` / `Cargo.toml` / `go.mod` / `pom.xml`

Extract: project name, description, language(s), framework(s).

### 2. Structure Map

List the top-level directory structure. Identify:
- **Monorepo?** Look for `nx.json`, `turbo.json`, `lerna.json`, `pnpm-workspace.yaml`, or multiple `package.json` files
- **Services/packages**: List each with a 1-line description inferred from its config or README
- **Key directories**: `src/`, `lib/`, `tests/`, `scripts/`, `infra/`, `k8s/`, `.github/`

### 3. Tech Stack

Discover from config files:
- **Languages & versions**: from config files, `.tool-versions`, `.nvmrc`, `Dockerfile`
- **Frameworks**: from dependencies (React, Next, Fastify, Django, etc.)
- **Databases**: from docker-compose, env files, ORM config
- **Testing**: from test config (vitest, jest, pytest, go test, etc.)
- **Linting/formatting**: from eslint, prettier, ruff, black configs
- **CI/CD**: from `.github/workflows/`, `Jenkinsfile`, `.gitlab-ci.yml`

### 4. Key Commands

Discover how to build, test, and lint:
- Check `Makefile` targets
- Check `package.json` scripts
- Check `pyproject.toml` scripts
- Check `Dockerfile` / `docker-compose.yml` for services
- Check `.factory/services.yaml` for project-managed services, if present

Present as a table: `| Action | Command |`

### 5. Project Agent Assets

Check for project-level agent configuration, preferring `.agents/` but also inspecting
legacy `.factory/` locations when present:
- `.agents/skills/` or `.factory/skills/` -- list skills with descriptions
- `.agents/sub-agents/` or `.factory/droids/` -- list sub-agents/droids
- `.agents/commands/` or `.factory/commands/` -- list commands
- `.agents/AGENTS.md` or `.factory/AGENTS.md` -- summarize key conventions
- `.factory/services.yaml` -- list services if present

### 6. Git Context

Run:
- `git remote get-url origin` -- repo location
- `git branch --show-current` -- current branch
- `git log --oneline -5` -- recent activity
- `git symbolic-ref refs/remotes/origin/HEAD` -- default branch

### 7. Output

Present a structured summary:

```
## [Project Name]

**Stack**: [languages] / [frameworks] / [databases]
**Structure**: [monorepo|single-service|library] with [N] packages/services
**Default branch**: [branch]

### Services
| Service | Path | Description |
|---------|------|-------------|
| ... | ... | ... |

### Key Commands
| Action | Command |
|--------|---------|
| Install | ... |
| Test | ... |
| Lint | ... |
| Build | ... |
| Dev server | ... |

### Conventions
- [Key conventions from AGENTS.md]

### Agent Assets
- [N] project skills, [N] sub-agents/droids, [N] commands
- [Notable items]

### Recent Activity
- [Last 5 commits]
```

## Notes

- Don't read every file -- skim config files and READMEs for the summary
- If the project is very large (50+ top-level dirs), group by category rather than listing all
- Flag anything unusual: missing tests, no CI, no README, etc.
