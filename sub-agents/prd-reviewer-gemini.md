---
name: prd-reviewer-gemini
description: Reviews product requirements documents (PRDs) for completeness, consistency, and quality (Gemini 3.1 Pro)
model: gemini-3.1-pro
reasoningEffort: high
tools: ["Read", "Grep", "Glob", "WebSearch"]
readonly: true
---
@@include shared/prd-review.md
