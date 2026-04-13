---
name: code-reviewer-opus
description: Code reviewer for PR review passes (Opus 4.6)
model: claude-opus-4-6
reasoningEffort: max
tools: ["Read", "Grep", "Glob", "LS"]
readonly: true
---
@@include shared/code-review.md
