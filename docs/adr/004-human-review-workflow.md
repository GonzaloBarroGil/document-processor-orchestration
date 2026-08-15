# ADR 004 — Human-in-the-Loop Review Workflow

**Status:** Accepted
**Date:** 2026-08-14

## Context

`docs/feasibility.md` §11.3 left the manual-review workflow as an open question. The approved
product spec (F12) and contract resolve it: reviewers must be able to correct and approve extracted
fields so bad extractions do not propagate downstream. This ADR pins down the mechanics.

## Decision

A dedicated review workflow: a review queue of documents needing attention, and a manual
`approve` / `reject` / `request_changes` action that persists edited fields.

## Details

- `GET /review/queue` lists documents awaiting review (`VALIDATION_FAILED` or manual-fix flagged),
  paginated, Bearer-only (`REVIEWER`/`ADMIN`).
- `PATCH /documents/{id}/review` accepts an `action` (`approve`, `reject`, `request_changes`),
  optional `edited_fields` (manual-fix overrides), and an optional `comment`.
- `approve` persists `edited_fields` and marks the document `reviewed` (with `reviewed_by`,
  `reviewed_at`); `request_changes` flags the document for re-extraction.
- Every review action writes an immutable `extraction_audit` row (D8, ADR 005 observability).

## Alternatives Considered

| Option | Pros | Cons |
|--------|------|------|
| Auto-approve all validation failures | No UI work | Defeats the purpose of HITL |
| Separate review microservice | Clean boundary | Unneeded scale; violates v1 scope |
| **Review endpoints on the existing service** | Simple, reuses auth + audit | More endpoints in one service |

## Consequences

- `documents` gains `reviewed`, `reviewed_by`, `reviewed_at`, `edited_fields` (plan §3).
- The web app ships a review-queue + manual-fix editor (task A1.4).
