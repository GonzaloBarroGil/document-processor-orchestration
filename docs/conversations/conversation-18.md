# Conversation 18 — A0.3 auth session store + refresh interceptor

> **Date:** 2026-08-20
> **Source session:** A0.3 (auth session store + refresh interceptor) in `document-processor-web`
> **Outcome:** token lifecycle (login/refresh/logout), session store, and refresh interceptor
> implemented and tested. Pending HITL commit.

## Summary

Implemented the auth token lifecycle for the web app: a localStorage-backed session store, typed
`login`/`refreshTokens`/`logout` functions consuming the generated `apiClient`, and an
`openapi-fetch` middleware that attaches the Bearer token and auto-refreshes (with de-duplication)
on 401 before retrying. Wired the middleware in `main.tsx`.

## Decisions

| ID | Decision | Status |
|----|----------|--------|
| — | Session persisted in `localStorage` (survives reload); keys `document-processor.access-token` / `.refresh-token` | Applied |
| — | Refresh interceptor as `openapi-fetch` middleware; dedupes concurrent refreshes via a shared promise; skips `/auth/login` + `/auth/refresh`; retries via raw `fetch(request.clone())` to avoid recursion | Applied |
| — | `createAuthMiddleware()` factory (testable) + idempotent `attachAuthMiddleware()` | Applied |
| — | **Fix:** Node 25 ships an experimental `localStorage` global (broken without `--localstorage-file`) that shadowed jsdom's; added an in-memory `Storage` polyfill in `test-setup.ts` (test-only) | Applied |
| — | `baseUrl` remains `VITE_API_URL ?? http://localhost:8000` | Applied |

## Deliverables

- `document-processor-web/src/auth/{session,auth,middleware}.ts`.
- `document-processor-web/src/auth/{session,auth,middleware}.test.ts` (11 tests).
- `document-processor-web/src/main.tsx` — attaches the middleware at startup.
- `document-processor-web/src/test-setup.ts`, `vite.config.ts` — localStorage polyfill + jsdom URL.

## Verification

- `pnpm typecheck` — clean.
- `pnpm lint` — clean.
- `pnpm test -- --run` — 12 passed (4 files).

## Transcript (condensed)

**user** — Proceed with A0.3.

**assistant** — Implemented the session store, login/refresh/logout functions, and the refresh
interceptor; fixed the Node 25 localStorage shadowing issue; verified all green.

> **Note:** nothing committed by the agent — awaiting HITL commit per governance §9. Next: **A1.1**
> (login view + route guards + role-aware nav).
