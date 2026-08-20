# Conversation 13 — W1.3 review service + W1.4 review endpoints

> **Date:** 2026-08-20
> **Source session:** W1.3 (review service) + W1.4 (review endpoints) in `document-processor`
> **Outcome:** review workflow implemented, wired behind RBAC, and verified against Postgres 16.
> Pending HITL commit.

## Summary

Implemented the human-in-the-loop review workflow: a `ReviewService` that applies
`approve`/`reject`/`request_changes` decisions to documents, and two Bearer-gated endpoints
(`PATCH /documents/{id}/review`, `GET /review/queue`). Extended the `Document` domain model and
repository with the review fields, and updated the API-key middleware to recognize Bearer-only
review paths (prefix matching, since the review PATCH path is dynamic).

## Decisions

| ID | Decision | Status |
|----|----------|--------|
| — | Review semantics: `approve` → `reviewed=True` + `COMPLETED`; `reject` → `reviewed=True` + `VALIDATION_FAILED`; `request_changes` → `reviewed=False` + `PENDING` (re-queued for re-extraction) | Applied |
| — | Review queue = `reviewed == False AND status IN (COMPLETED, VALIDATION_FAILED)` | Applied |
| — | `ReviewService` is a separate service (per plan) sharing the same `DocumentRepositoryPort` as `DocumentService` | Applied |
| — | `create_app` gains an optional `review_service` param (mirrors `auth_service`) | Applied |
| — | Middleware bearer detection refactored to prefix matching for `/api/v1/review/*` and `/api/v1/documents/*/review` (dynamic path could not be exact-matched) | Applied |
| — | `comment` accepted in `ReviewRequest` but not persisted yet — reserved for W1.7 audit trail | Applied |
| — | Contract `Document` schema omits `reviewed`/`reviewed_by`/`reviewed_at`/`edited_fields`; response includes them (non-breaking) — schedule a contract update | Deferred |

## Deliverables

- `domain/models/document.py` — `ReviewAction` enum + `user_id`/`reviewed`/`reviewed_by`/
  `reviewed_at`/`edited_fields` on `Document`.
- `domain/ports/document_repository.py` — `update_review`, `list_review_queue`.
- `domain/services/review_service.py` — new.
- `adapters/persistence/postgresql/repository.py` — `update_review` + `list_review_queue`.
- `adapters/persistence/postgresql/models.py` — converter updates; `edited_fields` typed
  `dict[str, str]`.
- `adapters/web/api/review.py` — new router (2 endpoints, `require_reviewer`).
- `adapters/web/api/deps.py` — `set/get_review_service`; `main.py` — wiring; `middleware/auth.py`
  — bearer prefix matching.
- Tests: `tests/unit/domain/test_review_service.py`, `tests/unit/adapters/test_repository.py`
  (extended), `tests/integration/test_review_api.py`.

## Verification

- `mypy -p document_processor` — clean (80 files).
- `ruff check src tests` — clean; `ruff format` — clean.
- `pytest tests/unit tests/bdd` — 142 passed; `pytest tests/integration` — 34 passed.
- Real-Postgres check: queue lists unreviewed `VALIDATION_FAILED`+`COMPLETED` → approve persists
  fields/status → doc leaves queue; `reviewed_by` FK enforced.

## Transcript (condensed)

**user** — Proceed (W1.3/W1.4).

**assistant** — Implemented the review service + endpoints, wired RBAC, refactored the middleware
for dynamic bearer paths. Verified statically, via the mock-based integration suite, and against
real Postgres (where the FK on `reviewed_by` was confirmed).

> **Note:** nothing committed by the agent — awaiting HITL commit per governance §9. Next tasks:
> **W1.5** (export endpoint), **W1.6** (daily quota), **W1.7** (audit/dead-letter wiring).
