# Conversation 21 — A1.3 document detail view

> **Date:** 2026-08-20
> **Source session:** A1.3 (document detail view) in `document-processor-web`
> **Outcome:** F11 detail view (metadata, parsed fields, confidence, validation) implemented. Pending HITL commit.

## Summary

Implemented the document detail view: a `useDocument` hook over
`GET /api/v1/documents/{document_id}` and a `DocumentDetailView` rendering metadata, parsed fields
(confidence + key/value table + raw text), the validation result, and any error detail. Linked each
row of the document list to the detail page and added the `/documents/:documentId` route.

## Decisions

| ID | Decision | Status |
|----|----------|--------|
| — | `useDocument` mirrors `useDocuments` (plain hook, generated-client types) | Applied |
| — | Detail route `/documents/:documentId`; list rows link via `Link` to it | Applied |
| — | **Fix:** adding the `Link` to the list broke its tests (router context required); wrapped the list tests in `MemoryRouter` | Applied |

## Deliverables

- `document-processor-web/src/features/documents/use-document.ts`.
- `document-processor-web/src/features/documents/document-detail-view.tsx`.
- `document-processor-web/src/features/documents/{use-document.test.ts,document-detail-view.test.tsx}`.
- `document-processor-web/src/features/documents/document-list-view.tsx` — row link.
- `document-processor-web/src/App.tsx` — detail route.

## Verification

- `pnpm typecheck` — clean.
- `pnpm lint` — clean (0 warnings).
- `pnpm test -- --run` — 30 passed (12 files).
- `pnpm build` — builds.

## Transcript (condensed)

**user** — Proceed with A1.3.

**assistant** — Implemented the document detail hook and view, linked the list to the detail page,
and wrapped the list tests in a router context after adding the link.

> **Note:** nothing committed by the agent — awaiting HITL commit per governance §9. Next: **A1.4**
> (review queue + manual-fix editor).
