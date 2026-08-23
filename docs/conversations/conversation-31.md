# Conversation 31 — M1.5 sync engine

> **Date:** 2026-08-21
> **Source session:** M1.5 (sync) in `document-processor-mobile`
> **Outcome:** FIFO upload loop with backoff/dead-letter + status polling. M-stream core logic complete.
> Pending HITL commit.

## Summary

Implemented the sync engine: an `Uploader` port + `apiUploader` (multipart `POST /documents`), a
`SyncEngine` (FIFO loop, backoff, dead-letter), and a status-polling helper. This completes the
offline-queue core (F17).

## Decisions

| ID | Decision | Status |
|----|----------|--------|
| — | `body: formData as never` for the multipart upload (generated type types the binary `file` as `string`) | Applied |
| — | `SyncEngine` takes injectable `sleep` (so backoff is testable without real waits) | Applied |
| — | `pollUntilTerminal` takes injectable `poll` + `sleep` | Applied |
| — | **Blocker flagged:** "auth via JWT" requires the mobile app to have a session store + refresh middleware (reuse of web `auth/`) — no M-stream task allocates it | Deferred → next |

## Deliverables

- `document-processor-mobile/src/sync/{uploader.ts,status.ts,sync-engine.ts}`.
- Tests: `sync-engine.test.ts`, `status.test.ts`.

## Verification

- `pnpm typecheck` — clean; `pnpm lint` — clean; `pnpm test -- --run` — 24 passed (8 files); `pnpm build` — builds.

## Transcript (condensed)

**user** — Proceed with M1.5.

**assistant** — Implemented the uploader, status polling, and sync engine; verified green. Flagged
the auth reuse as the remaining blocker before M2.

> **Note:** nothing committed by the agent — awaiting HITL commit per governance §9. Next: **auth
> reuse** (mobile session store + refresh middleware), then M2 test tasks.
