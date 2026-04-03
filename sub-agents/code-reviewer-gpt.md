---
name: code-reviewer-gpt
description: Post-builder code reviewer with auto-fix capability (GPT 5.4)
model: gpt-5.4
reasoningEffort: xhigh
tools: ["Read", "Grep", "Glob", "LS"]
readonly: true
---
@@include shared/code-review.md
