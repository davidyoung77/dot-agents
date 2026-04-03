# Review Diversity Pattern

When asked to perform a code review, spec review, or PRD review, prefer the
dedicated review workflow if one exists (for example, use `pr-review` for pull
requests). When direct reviewer fan-out is appropriate, use capability-aware
review diversity:

- On platforms that support distinct model assignment per reviewer, run the full
  reviewer family in parallel:
  - **Code review**: `code-reviewer-opus`, `code-reviewer-gpt`, `code-reviewer-gemini`
  - **Spec review**: `spec-reviewer-opus`, `spec-reviewer-gpt`, `spec-reviewer-gemini`
  - **PRD review**: `prd-reviewer-opus`, `prd-reviewer-gpt`, `prd-reviewer-gemini`
- On platforms that do not support true per-reviewer model selection, keep the
  reviewer family only if the prompt/lens diversity still adds value. Treat the
  results as prompt-diverse passes, not true cross-model consensus, and say so
  explicitly in the summary.
- If parallel fan-out adds little value for the task, one strong pass plus a
  targeted second lens is acceptable.

After the reviewer passes return, synthesize the results:
1. Merge overlapping findings (deduplicate)
2. Flag disagreements — where one reviewer flags an issue but others don't, call it out
3. Prioritize by consensus, while weighting true cross-model agreement higher than prompt-only agreement

If a reviewer pass times out or fails, proceed with the remaining signal when it
is still sufficient.

This rule does NOT apply when a specific reviewer is invoked by name. In that
case, run only the named reviewer.
