---
description: Run the governance-repo validation suite and enforce the conversation-persistence gate (D10)
agent: validator
subtask: true
---

Run the governance-repo validation suite:

1. Markdown lint on `docs/` (if configured)
2. Link check on `docs/` (no broken relative links)
3. OpenAPI lint (`spectral`) on `docs/contracts/` (if contracts exist)
4. Glossary consistency check

Then enforce the conversation-persistence gate (feasibility §8.6 / decision D10):

- Verify `docs/conversations/conversation-N.md` exists and contains a raw transcript
  plus a decisions table.
- **The orchestrating agent** is responsible for writing that file before this gate
  can pass (subagents run in fresh contexts and cannot see the transcript).
- A missing or empty conversation file blocks the gate: report FAIL with the path.

Report results as a structured validation report ending with the final gate:
`Conversation persisted? (path)` — PASS/FAIL.
