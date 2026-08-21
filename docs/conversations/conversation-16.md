# Conversation 16 — A0.1 web-app scaffold

> **Date:** 2026-08-20
> **Source session:** A0.1 (Vite + React + TS scaffold) in `document-processor-web`
> **Outcome:** runnable SPA skeleton with lint/typecheck/test/build green. Pending HITL commit.

## Summary

Started the A stream (web app). Scaffolded `document-processor-web` per the plan §4 stack:
React 19 + Vite 6 + TypeScript (strict), pnpm, ESLint 9 (flat config), Vitest 3 + React Testing
Library + jsdom. The `.opencode/` SDD toolset and contract-check CI were already in place from the
feasibility §13 step 3 scaffolding.

## Decisions

| ID | Decision | Status |
|----|----------|--------|
| — | Package manager pnpm (plan-mandated); installed `pnpm@11.22.0` globally (was missing) | Applied |
| — | pnpm 11 renamed build-script approval to `allowBuilds`; declared `esbuild` in `pnpm-workspace.yaml` so its postinstall runs | Applied |
| — | ESLint 9 flat config (`typescript-eslint` + `react-hooks` + `react-refresh`) | Applied |
| — | Vitest `globals: true` + `@testing-library/jest-dom/vitest` setup (matches cesar-web conventions) | Applied |
| — | `generate:client` script declared but deps deferred to A0.2 (`openapi-typescript`) | Applied |

## Deliverables

- `document-processor-web/package.json`, `tsconfig.json`, `vite.config.ts`, `eslint.config.js`,
  `pnpm-workspace.yaml`, `.gitignore`.
- `document-processor-web/index.html`, `src/main.tsx`, `src/App.tsx`, `src/test-setup.ts`,
  `src/App.test.tsx`.

## Verification

- `pnpm lint` — clean.
- `pnpm typecheck` — clean.
- `pnpm test -- --run` — 1 passed.
- `pnpm build` — builds (28 modules).

## Transcript (condensed)

**user** — Proceed (A0.1).

**assistant** — Scaffolded the Vite + React + TS app; resolved pnpm 11's build-approval for esbuild;
verified lint/typecheck/test/build all green.

> **Note:** nothing committed by the agent — awaiting HITL commit per governance §9. Next: **A0.2**
> (generate typed client from `openapi.yaml`).
