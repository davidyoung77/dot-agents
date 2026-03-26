# Personal Agent Instructions

## Git Workflow
- Always run tests/lint before suggesting commits
- Use conventional commits with scope: `type(scope): message`
- Never force push without explicit permission

## Code Style Preferences
- Keep code concise and readable
- Prefer readable code; concise one-liners are fine when clearer than verbose alternatives
- Add comments only when the "why" isn't obvious

## Modularity First
Before implementing any new feature or pattern:
- If the same logic would need to exist in 2+ places, extract it into a shared module/component/hook FIRST
- Don't inline something then refactor later -- design the reusable abstraction up front
- When adding behavior to UI components, prefer wrapper/context patterns over copy-pasting integration code into each screen
- Ask "will another screen/component need this?" before writing it inline

## Memory
Preferences, decisions, and gotchas are in `~/.agents/memories.md`.
Check this file before making style, architecture, or tooling decisions.
When you learn something worth remembering (a gotcha, a preference, an architecture decision),
update the file. Keep it under ~30 lines — prune entries that are already captured in config files.
