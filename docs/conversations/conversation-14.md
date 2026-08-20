# Conversation 14 — W1.5 export + W1.6 quota + W1.7 audit/dead-letter

> **Date:** 2026-08-20
> **Source session:** W1.5 (export), W1.6 (daily quota), W1.7 (audit + dead-letter) in `document-processor`
> **Outcome:** export endpoint, daily-quota enforcement, and audit/dead-letter wiring implemented and
> verified against Postgres 16. Pending HITL commit.

## Summary

Closed the remaining additive W-stream features. W1.5 adds a JSON/CSV export endpoint; W1.6 adds the
D12 daily quota (atomic `daily_usage` upsert, enforced on ingest with a worker safety-net); W1.7
wires the immutable `extraction_audit` trail and `failed_extractions` dead-letter into the pipeline
and review flow. Real-Postgres verification surfaced and fixed a pre-existing `model_dump()`
datetime-serialization bug in the persist path.

## Decisions

| ID | Decision | Status |
|----|----------|--------|
| — | Export: `ExportService` (separate, mirrors `ReviewService`); JSON default, CSV via `Accept: text/csv`; X-API-Key auth (consistent with existing documents endpoints; Bearer "either" arm deferred) | Applied |
| — | Quota: atomic `INSERT … ON CONFLICT … RETURNING` upsert on `daily_usage`; ingest enforces `global` + `key:<prefix>` scopes (429 + `Retry-After` until midnight); worker gets a `quota_check` safety-net in `poll_queue` | Applied |
| — | Audit/dead-letter: `AuditPort` + `FailedExtractionPort` are optional constructor params on `DocumentService`/`ReviewService` (existing fixtures untouched); OCR adapters expose `provider` | Applied |
| — | Audit actions: `OCR`/`VALIDATE` (pipeline) and `REVIEW_APPROVE`/`REVIEW_REJECT`/`REVIEW_REQUEST_CHANGES` (review) | Applied |
| — | **Bug fix:** `update_parsed_data` + `document_to_model` used `model_dump()` (leaked `datetime` into JSONB); changed to `model_dump(mode="json")` | Applied |
| — | Worker dequeue "before dequeuing" quota enforcement: ingest (atomic reserve) is primary; worker check is a strict-over-cap safety net | Applied |

## Deliverables

- `domain/models/export.py`, `domain/services/export_service.py`, `adapters/web/api/export.py` (W1.5).
- `domain/ports/daily_usage.py`, `domain/services/quota_service.py`,
  `adapters/persistence/postgresql/daily_usage_repository.py`; `daily_document_cap` config;
  ingest + `poll_queue` enforcement (W1.6).
- `domain/models/audit.py`, `domain/ports/audit.py`, `domain/ports/failed_extraction.py`,
  `adapters/persistence/postgresql/{audit,failed_extraction}_repository.py`; `provider` on OCR
  adapters; `DocumentService`/`ReviewService` wiring (W1.7).
- Tests: `test_export_service.py`, `test_quota_service.py`, `test_daily_usage_repository.py`,
  `test_audit_repositories.py`, `test_document_service_audit.py`, `test_review_service.py`
  (extended), `test_export_api.py`, `test_quota_api.py`.

## Verification

- `mypy -p document_processor` — clean (91 files).
- `ruff check src tests` — clean; `ruff format --check src` — clean.
- `pytest tests/unit tests/bdd` — 159 passed; `pytest tests/integration` — 40 passed.
- Real-Postgres checks: quota increments to cap then raises; audit rows (OCR+VALIDATE) and
  dead-letter rows persist correctly.

## Transcript (condensed)

**user** — Proceed (W1.5/W1.6/W1.7).

**assistant** — Implemented export, daily quota, and audit/dead-letter wiring; real-Postgres checks
caught and fixed a datetime-serialization bug in the persist path.

> **Note:** nothing committed by the agent — awaiting HITL commit per governance §9. Remaining W-stream:
> **W1.8** (auth/review/export/quota BDD features), **W1.9** (integration tests), **W1.10** (worker E2E).
