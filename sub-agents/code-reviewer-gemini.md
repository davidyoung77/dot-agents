---
name: code-reviewer-gemini
description: Post-builder code reviewer with auto-fix capability (Gemini 3.1 Pro)
model: gemini-3.1-pro-preview
reasoningEffort: high
tools: ["Read", "Grep", "Glob", "LS"]
readonly: true
---
@@include shared/code-review.md
