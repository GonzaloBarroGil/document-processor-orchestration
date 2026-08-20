# Conversation 15 — W1.8/W1.9/W1.10 (BDD + real-DB + E2E) — W stream complete

> **Date:** 2026-08-20
> **Source session:** W1.8 (BDD), W1.9 (real-DB integration), W1.10 (worker E2E) in `document-processor`
> **Outcome:** web-service stream (W0.1–W1.10) fully delivered. Pending HITL commit.

## Summary

Finished the W stream. W1.8 added acceptance BDD features for JWT auth, review, export, and the
daily quota (spec F10/F12/F13/F18). W1.9 added real-DB (testcontainers) integration tests for the
auth/review/export flows. W1.10 added the full-pipeline worker E2E (ingest → OCR → validate →
audit → review). Real-DB verification surfaced and fixed a test-isolation bug in the service
registry.

## Decisions

| ID | Decision | Status |
|----|----------|--------|
| — | BDD: extend the shared `bdd_test_client` to a full client (auth + review + export + quota services, seeded `REVIEWER` user) rather than per-feature fixtures | Applied |
| — | BDD: unique step wording per new feature (avoids pytest-bdd duplicate-step collisions with existing steps) | Applied |
| — | `_REVIEWER_PASSWORD_HASH` computed once at conftest import (argon2 hash reused across tests) | Applied |
| — | **Bug fix:** `_quota_service` module-global leaked across `create_app` calls (stale quota mock caused false 429s in unrelated rate-limit BDD tests); `create_app` now always calls `set_quota_service(quota_service)` (including None) | Applied |
| — | W1.9/W1.10 use `PostgresContainer` + `Base.metadata.create_all` (mirrors `test_worker.py`) | Applied |

## Deliverables

- `tests/bdd/features/{jwt_authentication,manual_review,export,daily_quota}.feature` + matching
  step files (`test_{jwt_auth,review,export,quota}_steps.py`).
- `tests/bdd/conftest.py` — full-client fixtures (`bdd_user`, `bdd_mock_usage_repo`, all services).
- `tests/integration/test_auth_db.py`, `test_review_export_db.py`, `test_pipeline_e2e.py`.
- `src/.../web/api/deps.py` + `main.py` — quota-service reset fix.

## Verification

- `mypy -p document_processor` — clean (91 files).
- `ruff check src tests` — clean; `ruff format --check src` — clean.
- `pytest tests/unit tests/bdd` — 168 passed (was 159, +9 scenarios).
- `pytest tests/integration` — 45 passed (was 40, +5 real-DB).

## Transcript (condensed)

**user** — Proceed (W1.8/W1.9/W1.10).

**assistant** — Added BDD features, real-DB integration tests, and the pipeline E2E; fixed a
test-isolation bug where the module-global quota service leaked between `create_app` calls.

> **Note:** nothing committed by the agent — awaiting HITL commit per governance §9. **W stream
> complete.** Next: **A stream** (`document-processor-web`) or remaining X-stream follow-ups.
