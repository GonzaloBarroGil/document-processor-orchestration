# Conversation 25 — A2.1 data-fetch integration tests (fetch-stub, not MSW)

> **Date:** 2026-08-21
> **Source session:** A2.1 (integration tests) in `document-processor-web`
> **Outcome:** integration tests exercising the real generated client via a fetch stub. Pending HITL commit.

## Summary

Delivered A2.1 with a pivot: MSW's fetch interceptor is incompatible with this environment, so I
wrote a minimal `mockApi` fetch stub that delivers the same value (testing the real `openapi-fetch`
client + hooks against mocked HTTP, rather than the mocked client). Also fixed a latent bug where
`openapi-fetch` captured `globalThis.fetch` at import time.

## Decisions

| ID | Decision | Status |
|----|----------|--------|
| — | **Root cause:** `openapi-fetch` captures `globalThis.fetch` at `createClient` time, so test-time fetch patches are ignored; fixed by making the client delegate to `globalThis.fetch` lazily at request time | Applied |
| — | MSW attempted first but its `@mswjs/interceptors` fetch interceptor is broken under vitest jsdom + Node 23/25; **pivoted to a `mockApi` fetch stub** (pathname+method router via `vi.stubGlobal`) | Applied |
| — | Method routing keyed off the `Request.method` (openapi-fetch passes a `Request`, not `init.method`) | Applied |
| — | `vi.unstubAllGlobals()` in `afterEach` (test-setup) | Applied |

## Deliverables

- `document-processor-web/src/test/mock-api.ts` — `mockApi` helper.
- `document-processor-web/src/api/client.ts` — lazy `fetch` delegation fix.
- `document-processor-web/src/test-setup.ts` — unstub globals.
- `document-processor-web/src/features/{documents,review,api-keys,dashboard}/*.integration.test.tsx` —
  5 integration tests.

## Verification

- `pnpm typecheck` — clean; `pnpm lint` — clean; `pnpm test -- --run` — 56 passed (24 files);
  `pnpm build` — succeeds.

## Transcript (condensed)

**user** — Proceed (A2.1).

**assistant** — Installed MSW, found its fetch interception broken in this environment (Node + jsdom),
fixed the openapi-fetch lazy-fetch issue, and pivoted to a minimal fetch stub; all green.

> **Note:** nothing committed by the agent — awaiting HITL commit per governance §9. Next: **A2.2**
> (Playwright E2E), or the **M stream** (mobile).
