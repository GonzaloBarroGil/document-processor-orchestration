# Conversation 30 — M1.3 backoff + M1.4 dead-letter

> **Date:** 2026-08-21
> **Source session:** M1.3 (backoff) + M1.4 (dead-letter) in `document-processor-mobile`
> **Outcome:** F17 retry policy modules. Pending HITL commit.

## Summary

Implemented the pure retry-policy modules: exponential backoff (`backoffDelayMs`) and dead-letter
(`shouldDeadLetter` + `retryDeadLetter`). These are the testable building blocks the M1.5 sync loop
will orchestrate.

## Decisions

| ID | Decision | Status |
|----|----------|--------|
| — | `backoffDelayMs(attempt) = min(base * 2^attempt, max)`, defaults 1s → 60s | Applied |
| — | `shouldDeadLetter(retryCount, maxRetries=5)`; `retryDeadLetter` resets status to `pending` + `retryCount: 0` | Applied |
| — | Kept the modules pure/independent — the FIFO upload loop that wires them lives in M1.5 | Applied |

## Deliverables

- `document-processor-mobile/src/queue/{backoff.ts,dead-letter.ts}`.
- Tests: `backoff.test.ts`, `dead-letter.test.ts`.

## Verification

- `pnpm typecheck` — clean; `pnpm lint` — clean; `pnpm test -- --run` — 18 passed (6 files).

## Transcript (condensed)

**user** — Proceed (M1.3/M1.4).

**assistant** — Implemented backoff and dead-letter modules with tests; verified green.

> **Note:** nothing committed by the agent — awaiting HITL commit per governance §9. Next: **M1.5**
> (sync engine: FIFO upload loop + backoff/dead-letter + status polling, JWT auth).
