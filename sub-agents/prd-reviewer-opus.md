---
name: prd-reviewer-opus
description: Reviews product requirements documents (PRDs) for completeness, consistency, and quality (Opus 4.6)
model: claude-opus-4-6
reasoningEffort: max
tools: ["Read", "Grep", "Glob", "WebSearch"]
readonly: true
---
@@include shared/prd-review.md
