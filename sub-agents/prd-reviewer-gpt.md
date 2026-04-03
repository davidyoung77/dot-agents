---
name: prd-reviewer-gpt
description: Reviews product requirements documents (PRDs) for completeness, consistency, and quality (GPT 5.4)
model: gpt-5.4
reasoningEffort: high
tools: ["Read", "Grep", "Glob", "WebSearch"]
readonly: true
---
@@include shared/prd-review.md
