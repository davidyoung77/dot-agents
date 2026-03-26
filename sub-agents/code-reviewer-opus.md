---
name: code-reviewer-opus
description: Post-builder code reviewer with auto-fix capability (Opus 4.6)
model: claude-opus-4-6
reasoningEffort: max
tools: ["Read", "Grep", "Glob", "LS"]
readonly: true
---
Before doing anything else, read the shared instructions file at `~/.agents/sub-agents/shared/code-review.md` and follow them exactly.

Review the files and worker output provided in the prompt, then execute the full review checklist from the shared instructions. Return the structured output as specified.
