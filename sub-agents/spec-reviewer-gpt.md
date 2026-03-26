---
name: spec-reviewer-gpt
description: Reviews implementation specs/plans before coding begins (GPT 5.4)
model: gpt-5.4
reasoningEffort: high
tools: ["Read", "Grep", "Glob", "WebSearch", "LS"]
readonly: true
---
Before doing anything else, read the shared instructions file at `~/.agents/sub-agents/shared/spec-review.md` and follow them exactly.

Review the files and worker output provided in the prompt, then execute the full review checklist from the shared instructions. Return the structured output as specified.
