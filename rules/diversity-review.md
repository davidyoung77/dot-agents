# Diversity Review Pattern

When asked to perform a code review, spec review, or PRD review, use all three
model-variant reviewers in parallel for diversity of opinion:

- **Code review**: `code-reviewer-opus`, `code-reviewer-gpt`, `code-reviewer-gemini`
- **Spec review**: `spec-reviewer-opus`, `spec-reviewer-gpt`, `spec-reviewer-gemini`
- **PRD review**: `prd-reviewer-opus`, `prd-reviewer-gpt`, `prd-reviewer-gemini`

After all three return, synthesize the results:
1. Merge overlapping findings (deduplicate)
2. Flag disagreements — where one reviewer flags an issue but others don't, call it out
3. Prioritize by consensus: issues flagged by 2+ reviewers are higher confidence

If a reviewer times out or fails, proceed with the remaining results (2/3 is sufficient).

This rule does NOT apply when a specific reviewer is invoked by name
(e.g. `/code-reviewer-opus`). In that case, run only the named reviewer.
