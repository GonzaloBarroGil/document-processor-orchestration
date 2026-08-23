# Conversation 32 — mobile auth reuse (JWT blocker)

> **Date:** 2026-08-21
> **Source session:** auth reuse in `document-processor-mobile`
> **Outcome:** mobile client is JWT-authenticated via the reused web auth module. Pending HITL commit.

## Summary

Closed the auth blocker flagged in M1.5. Reused the web app's `auth/{session,auth,middleware}.ts`
(identical content, since both clients expose the same `components`/`paths` types), copied their
three test files, and attached the refresh middleware in `main.tsx`. The mobile sync/upload is now
JWT-authenticated.

## Decisions

| ID | Decision | Status |
|----|----------|--------|
| — | Reuse (copy) the web `auth/` module rather than re-implement; identical client interface makes it drop-in | Applied |
| — | Only the token-lifecycle module (session/auth/middleware) reused; React UI (context/provider/login view) deferred until a mobile login screen is needed | Applied |
| — | `attachAuthMiddleware()` called in `main.tsx` | Applied |

## Deliverables

- `document-processor-mobile/src/auth/{session,auth,middleware}.ts`.
- `document-processor-mobile/src/auth/{session,auth,middleware}.test.ts`.
- `document-processor-mobile/src/main.tsx` — attach middleware.

## Verification

- `pnpm typecheck` — clean; `pnpm lint` — clean; `pnpm test -- --run` — 35 passed (11 files); `pnpm build` — builds.

## Transcript (condensed)

**user** — Update conversation and tackle the auth blocker before M2.

**assistant** — Reused the web auth module into mobile and attached the middleware; verified green.

> **Note:** nothing committed by the agent — awaiting HITL commit per governance §9. Next: **M2.1/M2.2/M2.3**
> (queue tests, plugin/IndexedDB integration, Maestro E2E).
