---
name: prd-reviewer-gemini
description: Reviews product requirements documents (PRDs) for completeness, consistency, and quality (Gemini 3.1 Pro)
model: gemini-3.1-pro-preview
reasoningEffort: high
tools: ["Read", "Grep", "Glob", "WebSearch"]
readonly: true
---
Before doing anything else, read the shared instructions file at `~/.agents/sub-agents/shared/prd-review.md` and follow them exactly.

Review the files and worker output provided in the prompt, then execute the full review checklist from the shared instructions. Return the structured output as specified.
