# Conversation 28 — M0.2 typed client generation (mobile)

> **Date:** 2026-08-21
> **Source session:** M0.2 (generate typed client) in `document-processor-mobile`
> **Outcome:** `openapi-fetch` client + `openapi-typescript` schema for the mobile app. Pending HITL commit.

## Summary

Generated the typed API client for the mobile app, mirroring the web app's A0.2: `openapi-fetch`
(runtime) + `openapi-typescript` (dev), `src/api/schema.d.ts` from the shared `openapi.yaml`, and a
`createClient<paths>` wrapper with the lazy `globalThis.fetch` delegation (the same fix from the
web app) and a `VITE_API_URL` override.

## Decisions

| ID | Decision | Status |
|----|----------|--------|
| — | Reuse the exact client pattern from the web app (same contract, same lazy-fetch fix) | Applied |
| — | Commit `schema.d.ts`; keep `openapi.yaml` gitignored | Applied |

## Deliverables

- `document-processor-mobile/package.json` — `openapi-fetch`, `openapi-typescript`.
- `document-processor-mobile/src/api/schema.d.ts` — generated types.
- `document-processor-mobile/src/api/client.ts` — `apiClient`.

## Verification

- `pnpm generate:client` — clean; `pnpm typecheck` — clean; `pnpm lint` — clean; `pnpm test -- --run` — 1 passed.

## Transcript (condensed)

**user** — Proceed with M0.2.

**assistant** — Generated the mobile typed client mirroring the web app; verified green.

> **Note:** nothing committed by the agent — awaiting HITL commit per governance §9. Next: **M1.1**
> (camera capture) and **M1.2** (offline queue store).
