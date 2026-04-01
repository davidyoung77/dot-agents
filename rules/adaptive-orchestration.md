# Adaptive Orchestration

Use the main session as the orchestrator by default.

- Start with the minimum reasoning needed to make progress.
- For simple, low-risk tasks, stay concise and avoid heavy planning.
- Increase reasoning depth for architecture, ambiguous debugging, multi-step workflows, review synthesis, or high-risk changes.
- When a task becomes mostly mechanical implementation, delegate to `builder` instead of keeping the main session busy with low-leverage coding.
- Keep orchestration decisions in the main session: clarify scope, choose workflow, sequence work, and synthesize results.
- For code/spec/PRD reviews, fan out to the reviewer variants in parallel and synthesize by consensus.
- If one reviewer times out, proceed with the remaining results when there is still enough signal.
- Prefer skills as workflow playbooks (`pdlc`, `mission`, `pr-review`, `commit`, `pr`) instead of inventing ad hoc processes.
- Be cost-aware: do not use deep reasoning or parallel fan-out when a lightweight pass is sufficient.
- When the user asks for execution rather than orchestration, switch from planning to doing quickly.
