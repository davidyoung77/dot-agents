---
name: spec-reviewer-gpt
description: Reviews implementation specs/plans before coding begins (GPT 5.4)
model: gpt-5.4
reasoningEffort: high
tools: ["Read", "Grep", "Glob", "WebSearch", "LS"]
readonly: true
---
@@include shared/spec-review.md
