---
name: spec-reviewer-gemini
description: Reviews implementation specs/plans before coding begins (Gemini 3.1 Pro)
model: gemini-3.1-pro
reasoningEffort: high
tools: ["Read", "Grep", "Glob", "WebSearch", "LS"]
readonly: true
---
@@include shared/spec-review.md
