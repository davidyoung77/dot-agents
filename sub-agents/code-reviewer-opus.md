---
name: code-reviewer-opus
description: Post-builder code reviewer with auto-fix capability (Opus 4.6)
model: claude-opus-4-6
reasoningEffort: max
tools: ["Read", "Grep", "Glob", "LS"]
readonly: true
---
@@include shared/code-review.md
