# Conversation 33 — M2.1/M2.2 mobile tests + coverage gate

> **Date:** 2026-08-21
> **Source session:** M2.1 (unit tests) + M2.2 (integration tests) in `document-processor-mobile`
> **Outcome:** queue/capture coverage ≥80% enforced; camera plugin + IndexedDB integration tests.
> Pending HITL commit.

## Summary

Added `@vitest/coverage-v8` with an 80% threshold over `src/{queue,sync,capture}/**`, and filled the
coverage gaps (uploader, camera gateway, status poll, sleep). Fixed a FIFO ordering bug in the queue
store. This completes M2.1 and M2.2.

## Decisions

| ID | Decision | Status |
|----|----------|--------|
| — | `@vitest/coverage-v8` pinned to `3.2.7` (the default resolved v4, mismatching vitest 3) | Applied |
| — | Coverage thresholds (80%) over `queue`/`sync`/`capture` | Applied |
| — | **Fix:** `QueueStore` used `Date.now()` for `createdAt` (collides within a ms → FIFO broke); replaced with monotonic `nextCreatedAt()` | Applied |
| — | Excluded `coverage/` from ESLint (generated reports tripped the "no warnings" gate) | Applied |

## Deliverables

- `document-processor-mobile/vite.config.ts` — coverage config.
- `document-processor-mobile/src/sync/uploader.test.ts`, `src/capture/camera-gateway.test.ts` (new).
- `document-processor-mobile/src/{queue/backoff,sync/status}.test.ts` (extended).
- `document-processor-mobile/src/queue/store.ts` — monotonic timestamp fix.
- `document-processor-mobile/eslint.config.js` — ignore `coverage`.

## Verification

- Coverage: 96.89% stmts / 86.48% branch / 87.5% funcs / 96.89% lines (gate passes).
- `pnpm test -- --run` — 41 passed (13 files); `pnpm typecheck`/`lint` clean; `pnpm build` — builds.

## Transcript (condensed)

**user** — Proceed (M2.1/M2.2).

**assistant** — Added coverage gate + tests, fixed the FIFO timestamp collision, verified green.

> **Note:** nothing committed by the agent — awaiting HITL commit per governance §9. Next: **M2.3**
> (Maestro/Detox E2E — capture → offline → sync; requires a native/emulator environment).
