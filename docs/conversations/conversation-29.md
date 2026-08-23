# Conversation 29 — M1.1 camera capture + M1.2 offline queue store

> **Date:** 2026-08-21
> **Source session:** M1.1 (camera capture) + M1.2 (queue store) in `document-processor-mobile`
> **Outcome:** F16 quality validation + capture wrapper, F17 IndexedDB queue store. Pending HITL commit.

## Summary

Implemented the mobile capture and queue foundations. M1.1 adds pure quality validation (min
1920×1080, JPEG, ≤5 MB) and a testable camera wrapper around `@capacitor/camera`. M1.2 adds an
IndexedDB-backed `QueueStore` with a 50-pending cap and FIFO ordering.

## Decisions

| ID | Decision | Status |
|----|----------|--------|
| — | `@capacitor/camera` pinned to `^7.0.0` (the default resolved v8, mismatching `@capacitor/core@7`) | Applied |
| — | Capacitor `Photo` has no width/height, so dimensions are read via an injectable `DimensionReader` (DOM `Image`), keeping `captureDocument` testable with a mock | Applied |
| — | Queue store uses `crypto`-free id generator; FIFO via a `createdAt` index | Applied |
| — | Queue tests use `fake-indexeddb` with a per-test `deleteDatabase` reset in `beforeEach` | Applied |

## Deliverables

- `document-processor-mobile/src/capture/{quality.ts,camera.ts}`.
- `document-processor-mobile/src/queue/{types.ts,store.ts}`.
- Tests: `quality.test.ts`, `camera.test.ts`, `store.test.ts`.
- `document-processor-mobile/src/test-setup.ts` — `fake-indexeddb/auto`.
- `document-processor-mobile/package.json` — `@capacitor/camera`, `fake-indexeddb`.

## Verification

- `pnpm typecheck` — clean; `pnpm lint` — clean; `pnpm test -- --run` — 13 passed; `pnpm build` — builds.

## Transcript (condensed)

**user** — Proceed (M1.1/M1.2).

**assistant** — Implemented camera quality validation + wrapper and the IndexedDB queue store;
pinned the camera plugin to v7; verified green.

> **Note:** nothing committed by the agent — awaiting HITL commit per governance §9. Next: **M1.3**
> (FIFO upload + exponential backoff) and **M1.4** (dead-letter + manual retry).
