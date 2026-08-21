# Conversation 19 — A1.1 login view + route guards + role-aware nav

> **Date:** 2026-08-20
> **Source session:** A1.1 (login view, route guards, role-aware nav) in `document-processor-web`
> **Outcome:** auth UI flow (F10) implemented with routing, guards, and RBAC-aware nav. Pending HITL commit.

## Summary

Implemented the front-end auth flow on top of A0.3. Added `react-router-dom@7` routing, an auth
context/provider (`AuthProvider` + `useAuth`), a `RequireAuth` route guard with an `adminOnly`
variant, a login view that redirects back to the intended route, and a role-aware nav (API Keys
link only for ADMIN). Added `getCurrentUser()` consuming the generated `/auth/me` type.

## Decisions

| ID | Decision | Status |
|----|----------|--------|
| — | `react-router-dom@7` for routing | Applied |
| — | `AuthProvider` (component) split from `AuthContext`/`useAuth` (non-component) to satisfy `react-refresh/only-export-components` ("no warnings" gate) | Applied |
| — | `RequireAuth` supports both layout (`<Outlet/>`) and wrapper (`children`) usage; `adminOnly` redirects non-ADMIN to `/` | Applied |
| — | `CurrentUser` derived from generated `components["schemas"]["User"]` (no hand-written DTOs) | Applied |
| — | Login redirects to `location.state.from` (fallback `/`) | Applied |

## Deliverables

- `document-processor-web/src/auth/{auth-context.ts,auth-provider.tsx,require-auth.tsx,login-view.tsx}`.
- `document-processor-web/src/components/{nav.tsx,app-layout.tsx}`.
- `document-processor-web/src/App.tsx` — route tree.
- `document-processor-web/src/api/client.ts` — exports `components`.
- Tests: `auth-context.test.tsx`, `require-auth.test.tsx`, `login-view.test.tsx`, `nav.test.tsx`.

## Verification

- `pnpm typecheck` — clean.
- `pnpm lint` — clean (0 warnings).
- `pnpm test -- --run` — 21 passed (8 files).
- `pnpm build` — builds (51 modules).

## Transcript (condensed)

**user** — Proceed (A1.1).

**assistant** — Implemented the auth UI (context/provider, guard, login, nav), resolved the
react-refresh warning by splitting the hook from the component, and verified everything green.

> **Note:** nothing committed by the agent — awaiting HITL commit per governance §9. Next: **A1.2**
> (document list + filters + pagination).
