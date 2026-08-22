# Conversation 22 — A1.4 review queue + manual-fix editor

> **Date:** 2026-08-20
> **Source session:** A1.4 (review UI) in `document-processor-web`
> **Outcome:** F12 review queue and manual-fix editor implemented. Pending HITL commit.

## Summary

Implemented the review workflow UI: a `useReviewQueue` hook over `GET /api/v1/review/queue`, a
`ReviewQueueView` listing documents awaiting review, a typed `submitReview` for
`PATCH /api/v1/documents/{id}/review`, and a `ReviewEditorView` with editable parsed fields, a
comment field, and Approve / Request changes / Reject actions. Wired `/review` and
`/review/:documentId` routes.

## Decisions

| ID | Decision | Status |
|----|----------|--------|
| — | Approve sends `edited_fields`; request_changes sends `comment` and stays on-page (shows "Changes requested"); approve/reject navigate back to `/review` | Applied |
| — | `ReviewAction` derived from generated `ReviewRequest["action"]` | Applied |
| — | **Fix:** test race — the editor syncs parsed fields into local state via `useEffect` after first render, so the test used async `findByLabelText` instead of sync `getByLabelText` | Applied |

## Deliverables

- `document-processor-web/src/features/review/{use-review-queue.ts,review-queue-view.tsx,submit-review.ts,review-editor-view.tsx}`.
- `document-processor-web/src/features/review/{use-review-queue.test.ts,review-queue-view.test.tsx,review-editor-view.test.tsx}`.
- `document-processor-web/src/App.tsx` — review routes.

## Verification

- `pnpm typecheck` — clean.
- `pnpm lint` — clean (0 warnings).
- `pnpm test -- --run` — 37 passed (15 files).
- `pnpm build` — builds.

## Transcript (condensed)

**user** — Proceed with A1.4.

**assistant** — Implemented the review queue and manual-fix editor; a test race surfaced and was
fixed by awaiting the field render.

> **Note:** nothing committed by the agent — awaiting HITL commit per governance §9. Next: **A1.5**
> (export view — JSON/CSV download).
