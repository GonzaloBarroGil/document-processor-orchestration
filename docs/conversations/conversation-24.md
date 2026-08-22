# Conversation 24 — F15 dashboard (contract + backend + A1.7 UI)

> **Date:** 2026-08-21
> **Source session:** F15 (dashboard) across the three repos
> **Outcome:** contract, backend stats endpoint, and web dashboard UI. Pending HITL commit.

## Summary

Closed the F15 gap (no stats endpoint existed). Added `GET /api/v1/dashboard` (Bearer) returning
counts-by-status plus recent activity, backed by a `count_by_status` repository query and a
`DashboardService`, and the A1.7 dashboard UI. This completes the A-stream feature tasks F10–F15.

## Decisions

| ID | Decision | Status |
|----|----------|--------|
| — | `DashboardSummary = { counts: dict[str,int], recent: Document[] }`; recent = first page of `list_documents` (created_at desc, size 10) | Applied |
| — | `DocumentRepositoryPort.count_by_status()` via SQL `GROUP BY status` | Applied |
| — | `/api/v1/dashboard` Bearer-only, `require_reviewer` (any authenticated user) | Applied |

## Deliverables

- `document-processor-orchestration/docs/contracts/openapi.yaml` — dashboard path + schema.
- `document-processor` — `DashboardSummary` model, `DashboardService`, `api/dashboard.py` router,
  `count_by_status` port + adapter, middleware bearer path, `deps.py`/`main.py` wiring.
- `document-processor-web` — regenerated client; `features/dashboard/` (`dashboard.ts`,
  `use-dashboard.ts`, `dashboard-view.tsx`); `/` route renders `DashboardView`.

## Verification

- Backend: mypy/ruff clean; `179` unit/BDD + `54` integration tests.
- Web: typecheck/lint clean; `51` tests; build succeeds.
- Contract: YAML parses (14 paths, 25 schemas).

## Transcript (condensed)

**user** — Proceed (F15).

**assistant** — Added the dashboard contract + backend stats endpoint + web UI end-to-end; verified
green. This completes the A-stream F10–F15 features.

> **Note:** nothing committed by the agent — awaiting HITL commit per governance §9. Next: **A2.1**
> (MSW integration tests) then **A2.2** (Playwright E2E), or the **M stream** (mobile).
