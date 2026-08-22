# Conversation 27 — M0.1 Capacitor + React scaffold

> **Date:** 2026-08-21
> **Source session:** M0.1 (mobile scaffold) in `document-processor-mobile`
> **Outcome:** runnable Capacitor + React skeleton. M stream started. Pending HITL commit.

## Summary

Scaffolded `document-processor-mobile`: Vite + React 19 + TypeScript (strict) + Capacitor 7,
mirroring the web app's tooling (pnpm, ESLint 9 flat config, Vitest 3 + RTL + jsdom) so web
components/hooks can be reused. The `.opencode/` SDD toolset and contract-check CI were already in
place from the feasibility §13 step 3 scaffolding.

## Decisions

| ID | Decision | Status |
|----|----------|--------|
| — | Capacitor 7 (`@capacitor/core` + `cli`/`ios`/`android`); `webDir: dist` | Applied |
| — | Mirror the web app (A0.1) tooling: same tsconfig/eslint/vitest + `MemoryStorage` polyfill, so web code is reusable | Applied |
| — | Native `ios`/`android` folders gitignored — generated on demand via `cap add` | Applied |
| — | `sync` + `generate:client` scripts declared; openapi deps deferred to M0.2 | Applied |

## Deliverables

- `document-processor-mobile/{package.json,tsconfig.json,vite.config.ts,eslint.config.js,capacitor.config.ts,pnpm-workspace.yaml,.gitignore}`.
- `document-processor-mobile/{index.html,src/main.tsx,src/App.tsx,src/test-setup.ts,src/App.test.tsx}`.

## Verification

- `pnpm typecheck` — clean; `pnpm lint` — clean; `pnpm test -- --run` — 1 passed; `pnpm build` — builds.
- `cap --version` — 7.6.8.

## Transcript (condensed)

**user** — Proceed with M0.1.

**assistant** — Scaffolded the Capacitor + React mobile app mirroring the web app conventions;
verified lint/typecheck/test/build green.

> **Note:** nothing committed by the agent — awaiting HITL commit per governance §9. Next: **M0.2**
> (generate the typed client from `openapi.yaml`).
