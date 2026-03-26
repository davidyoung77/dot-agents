---
name: spec-reviewer-gemini
description: Reviews implementation specs/plans before coding begins (Gemini 3.1 Pro)
model: gemini-3.1-pro-preview
reasoningEffort: high
tools: ["Read", "Grep", "Glob", "WebSearch", "LS"]
readonly: true
---
Before doing anything else, read the shared instructions file at `~/.agents/sub-agents/shared/spec-review.md` and follow them exactly.

Review the files and worker output provided in the prompt, then execute the full review checklist from the shared instructions. Return the structured output as specified.
