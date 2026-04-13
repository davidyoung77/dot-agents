---
name: code-reviewer-gemini
description: Code reviewer for PR review passes (Gemini 3.1 Pro)
model: gemini-3.1-pro
reasoningEffort: high
tools: ["Read", "Grep", "Glob", "LS"]
readonly: true
---
@@include shared/code-review.md
