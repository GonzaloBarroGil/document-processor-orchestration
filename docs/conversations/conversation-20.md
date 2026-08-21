# Conversation 20 — A1.2 document list + filters + pagination

> **Date:** 2026-08-20
> **Source session:** A1.2 (document list) in `document-processor-web`
> **Outcome:** F11 document browsing implemented with a typed data-fetching hook and list view.
> Pending HITL commit.

## Summary

Implemented the document browsing feature: a `useDocuments` data-fetching hook over
`GET /api/v1/documents` and a `DocumentListView` with status/type/region filters and pagination.
Wired the view into the `/documents` route.

## Decisions

| ID | Decision | Status |
|----|----------|--------|
| — | Plain `useEffect`/`useCallback` hook (no TanStack Query) — matches the plan's default `openapi-fetch` stack; can add a query library later | Applied |
| — | Hook exposes `documents`/`total`/`pages`/`loading`/`error`; deps are the primitive filter fields | Applied |
| — | Filter types derived from generated schema (`DocumentStatus`/`DocumentType`) — no hand-written DTOs | Applied |
| — | UI `""` ("All") maps to `undefined` before reaching the API query | Applied |

## Deliverables

- `document-processor-web/src/features/documents/use-documents.ts`.
- `document-processor-web/src/features/documents/document-list-view.tsx`.
- `document-processor-web/src/features/documents/{use-documents.test.ts,document-list-view.test.tsx}`.
- `document-processor-web/src/App.tsx` — routes `/documents` to `DocumentListView`.

## Verification

- `pnpm typecheck` — clean.
- `pnpm lint` — clean (0 warnings).
- `pnpm test -- --run` — 26 passed (10 files).
- `pnpm build` — builds (53 modules).

## Transcript (condensed)

**user** — Proceed (A1.2).

**assistant** — Implemented the document list hook and view with filters and pagination; verified
typecheck/lint/test/build green.

> **Note:** nothing committed by the agent — awaiting HITL commit per governance §9. Next: **A1.3**
> (document detail view).
