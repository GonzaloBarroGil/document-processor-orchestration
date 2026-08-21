# Conversation 17 — A0.2 typed client generation

> **Date:** 2026-08-20
> **Source session:** A0.2 (generate typed client) in `document-processor-web`
> **Outcome:** `openapi-fetch` client + `openapi-typescript` schema generated from the contract.
> Pending HITL commit.

## Summary

Generated the type-safe API client for the web app. Added `openapi-fetch` (runtime) and
`openapi-typescript` (dev), generated `src/api/schema.d.ts` from the hub `openapi.yaml`, and wired
a thin `createClient<paths>` wrapper in `src/api/client.ts` with a `VITE_API_URL` override.

## Decisions

| ID | Decision | Status |
|----|----------|--------|
| — | `openapi-typescript` + `openapi-fetch` (per feasibility §6.4), not `orval` | Applied |
| — | Commit the generated `schema.d.ts` (repo typechecks out-of-the-box); keep `openapi.yaml` gitignored (fetched in CI / copied locally) | Applied |
| — | Ignore `src/api/schema.d.ts` in ESLint (auto-generated) | Applied |
| — | `baseUrl` defaults to `http://localhost:8000`, overridable via `VITE_API_URL` | Applied |

## Deliverables

- `document-processor-web/package.json` — `openapi-fetch`, `openapi-typescript`; working
  `generate:client` script.
- `document-processor-web/src/api/schema.d.ts` — generated types (`paths`, `components`).
- `document-processor-web/src/api/client.ts` — `apiClient` + re-exported `paths`.
- `document-processor-web/eslint.config.js` — ignores the generated schema.

## Verification

- `pnpm generate:client` — clean.
- `pnpm typecheck` — clean (`createClient<paths>` resolves against generated types).
- `pnpm lint` — clean; `pnpm test -- --run` — 1 passed.

## Transcript (condensed)

**user** — Proceed (A0.2).

**assistant** — Generated the typed client from `openapi.yaml` and wired an `openapi-fetch`
wrapper; verified typecheck/lint/test green.

> **Note:** nothing committed by the agent — awaiting HITL commit per governance §9. Next: **A0.3**
> (auth session store + refresh interceptor + login/refresh hooks).
