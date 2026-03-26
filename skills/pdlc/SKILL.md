---
name: pdlc
description: |
  Product Development Lifecycle covering the full pipeline from requirements through architecture,
  planning, implementation, and validation. Tool-agnostic -- works with 8090, Jira, Linear,
  or plain markdown specs. Use when working on any structured software project.
---

# Product Development Lifecycle (PDLC)

## The Pipeline

Every project follows this pipeline. The discipline of completing each phase before the next is the value -- not any specific tool.

1. **Requirements** -- Define what to build and why
2. **Architecture** -- Define how to build it (blueprints/specs)
3. **Planning** -- Break architecture into work items with dependencies
4. **Implementation** -- Build it, following the specs
5. **Testing** -- Verify it works against real infrastructure
6. **Feedback** -- Collect user feedback, feed back into requirements

## Phase 1: Requirements

Define product overview, business problem, personas, success metrics, and feature tree.

**Artifacts produced:**
- Product overview (problem, personas, success metrics)
- Feature tree with parent/child hierarchy
- Technical requirements and constraints
- PRD documents per feature area

**Quality gate:** Run PRD reviewer droids (opus, gpt, gemini) for contradictions and completeness.

**Tools:** 8090 Refinery, Notion, Confluence, markdown PRDs in repo, Kiro specs

## Phase 2: Architecture (Blueprints)

Translate requirements into technical blueprints/specs. This is the most important phase -- blueprints are the source of truth for implementation.

**Artifacts produced:**
- Foundation blueprints: Backend, Frontend, Data Layer
- Feature blueprints: One per feature with detailed implementation specs
- Each blueprint contains: schemas, queries, API contracts, algorithms, data models

**Quality gate:** Run spec reviewer droids to catch cross-blueprint inconsistencies.

**Blueprint-first development:** Always consult blueprints before implementing. They should contain:
- Exact schema field definitions (Zod, TypeScript interfaces, etc.)
- Database query patterns
- REST/GraphQL endpoint request/response contracts
- Algorithm pseudocode with code examples
- Data flow diagrams and architecture decisions

**Tools:** 8090 Foundry, Kiro specs, GitHub spec-kit, architecture markdown in repo

## Phase 3: Planning

Break blueprints into work items organized into phases with dependencies.

**Artifacts produced:**
- Work items with: summary, scope, acceptance criteria, blueprint references
- Phases with dependency ordering
- Each work item traces back to a requirement and blueprint

**Patterns:**
- Tightly coupled items can be implemented together
- Phase ordering reflects dependencies -- don't skip ahead
- Items reference specific blueprints -- read those for implementation detail

**Tools:** 8090 Planner, Jira, Linear, GitHub Issues, plain markdown task lists

## Phase 4: Implementation

### Single Work Item Workflow
1. Read work item for scope and acceptance criteria
2. Read referenced blueprints for technical detail
3. Set status to in-progress
4. Implement following project AGENTS.md conventions
5. Run typecheck and unit tests
6. Commit with conventional commits: `feat(ITEM-ID): description`
7. Set status to completed

### Mission-Based Implementation
For bulk implementation (4+ interdependent items), use the **mission skill** (`~/.factory/skills/mission/SKILL.md`).

The mission skill handles prompt generation, work item status management, and post-mission review checklists -- agnostic to PM tool.

## Phase 5: Testing & Verification

### Unit tests are not enough
Unit tests mock infrastructure. Integration-level bugs only surface when running against real services. After every mission or significant implementation batch:

1. **Run reviewer droids** (code + spec + PRD) -- catches code-level issues
2. **Run smoke tests against real services** -- catches integration issues
3. **Fix findings** from both

**Bugs that pass unit tests but break at runtime:**
- Blocking Redis commands (BRPOP) on shared connections starving other operations
- Connection pool exhaustion under concurrent load
- Missing database indexes causing query hangs
- Cold-start timeouts from services blocking on first connection
- Stale database connections after infrastructure restarts

Code reviewers (even multi-model) consistently miss these because they reason about code structure, not runtime resource contention.

## Phase 6: Feedback Loop (Post-Deployment)

Collect user feedback from production, triage, and feed back into Phase 1 as new requirements or bug fixes.

**Tools:** 8090 Validator SDK, Sentry, LogRocket, custom feedback widgets, support tickets

---

## Key Practices

### Multi-Model Reviews
- Run parallel reviewer droids (opus, gpt, gemini) for code, spec, and PRD reviews
- Different models catch different issues -- run at least 2 in parallel
- Shared instruction files in `~/.factory/droids/shared/` keep reviewers consistent
- Reviewers + smoke tests are complementary, not substitutes

### Cost Control
- Use builder subagents for independent implementation tasks
- Batch related work items when they share the same files
- Delegate mechanical code generation to workers, keep orchestration decisions in main session

### Post-Mission Review Workflow
1. Run 3 code reviewers in parallel (opus, gpt, gemini)
2. Run 3 spec reviewers for blueprint drift (if blueprints exist)
3. Run 3 PRD reviewers for requirements coverage (if PRDs exist)
4. Fix any critical findings with worker subagents
5. Run smoke test against real services (Docker Compose, local dev, etc.)
6. Fix any runtime issues found during smoke test
7. Mark all work items as completed

---

## Tool-Specific: 8090 Software Factory

When using 8090 (factory.8090.ai), it maps to the pipeline as follows:

| Pipeline Phase | 8090 Module | What it does |
|---|---|---|
| Requirements | Refinery | Product overview, feature tree, PRD documents |
| Architecture | Foundry | Technical blueprints (foundation + feature-level) |
| Planning | Planner | Work orders organized into phases with dependencies |
| Feedback | Validator | Collects user feedback from production apps via JS SDK |

### 8090 MCP Tools

**Official Planner MCP** (HTTP at `https://api.factory.8090.dev/mcp/`):
- `list_work_orders` / `read_work_order` / `edit_work_order` / `search_work_orders`
- `list_blueprints` / `read_blueprint` / `search_blueprints`
- `list_requirements` / `read_requirement` / `search_requirements`
- `list_artifacts` / `read_artifact`

**Custom MCP** (~/Development/mcp/8090/):
- `list_projects` / `get_project_dashboard` / `get_blueprint_tree`
- `update_blueprint` -- PATCH blueprint content (Foundry API)
- `chat_with_agent` -- Send messages to Refinery/Foundry/Planner/Validator agents (SSE)

### 8090 API Details
- Auth: AWS Cognito accessToken as Bearer token (for custom MCP)
- Foundry blueprints: PATCH via `/foundry/projects/:id/blueprintsv2/:blueprintId`
- Agent chat: SSE streaming with content deltas
- No public API docs -- custom MCP was reverse-engineered from browser network calls

### 8090 Strengths
- Enforces pipeline discipline -- can't skip requirements and jump to code
- Non-engineers (PMs, architects) can edit requirements and review blueprints in a browser without git
- Blueprints as first-class queryable artifacts with MCP access
- Work order descriptions with linked blueprints and requirements give full context

### 8090 Limitations
- AI agents are weak compared to external agents (Factory Droid, Cursor)
- No Jira/Linear integration -- work orders are siloed
- No public API docs beyond Planner MCP
- No bulk export for blueprints or requirements
