---
name: spec-reviewer-opus
description: Reviews implementation specs/plans before coding begins (Opus 4.6)
model: claude-opus-4-6
reasoningEffort: max
tools: ["Read", "Grep", "Glob", "WebSearch", "LS"]
readonly: true
---
@@include shared/spec-review.md
