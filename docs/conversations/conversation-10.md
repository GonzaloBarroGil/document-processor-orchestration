# Conversation 10 — W0.3/W0.4/W0.5 schema migrations

> **Date:** 2026-08-19
> **Source session:** W0.3–W0.5 (users/refresh/audit/dead-letter/usage + documents delta) in `document-processor`
> **Outcome:** five new ORM models + four Alembic migrations, verified end-to-end against Postgres 16.
> Pending HITL commit.

## Summary

After W0.2 (auth services), proceeded to the W-stream migration tasks. Added the `users`,
`refresh_tokens`, `extraction_audit`, `failed_extractions`, and `daily_usage` tables plus the
`documents` review/ownership delta to `models.py`, and produced a complete Alembic migration
chain. A baseline migration was required because `migrations/versions/` was empty — the project
had Alembic scaffolding but had relied on `Base.metadata.create_all` (tests) with no migrations.

## Decisions

| ID | Decision | Status |
|----|----------|--------|
| — | Introduce a **baseline migration (0001)** for the pre-existing `documents`/`api_keys`/`storage_alerts` tables; otherwise the W0.5 `ALTER documents` could not chain | Applied |
| — | Migration chain: `0001` baseline → `0002` W0.3 (users+refresh_tokens) → `0003` W0.4 (audit+failed+daily) → `0004` W0.5 (documents delta) | Applied |
| — | Follow existing ORM convention (Python-side `default=`) for new-table defaults; no `server_default` — except `documents.reviewed`, which uses `server_default=text("false")` so the NOT NULL column can be added to an existing table | Applied |
| — | Index `idx_documents_user` uses `(user_id, created_at ASC)` instead of the plan's `DESC` — Postgres scans it backward for `ORDER BY ... DESC`; keeps ORM and migration byte-identical for `alembic check` | Applied |
| — | New id columns use `default=uuid4` (adapters such as `RefreshTokenRepositoryPort.save` do not supply an id) | Applied |

## Deliverables

- `document-processor/src/document_processor/adapters/persistence/postgresql/models.py` —
  `UserModel`, `RefreshTokenModel`, `ExtractionAuditModel`, `FailedExtractionModel`,
  `DailyUsageModel`; `DocumentModel` gains `user_id`, `reviewed`, `reviewed_by`, `reviewed_at`,
  `edited_fields` and `idx_documents_user`.
- `document-processor/.../migrations/versions/` —
  `0001_baseline.py`, `0002_users_refresh_tokens.py`, `0003_extraction_audit_failed_daily.py`,
  `0004_documents_review_delta.py`.

## Verification

- `alembic upgrade head` on Postgres 16 (testcontainers) — all 4 migrations apply cleanly.
- `alembic check` — "No new upgrade operations detected" (ORM/migrations in sync, zero drift).
- `mypy -p document_processor` — clean (75 files).
- `ruff check src` — clean; `ruff format --check src` — clean.
- `pytest tests/unit tests/bdd` — 116 passed; `pytest tests/integration` — 18 passed.

## Transcript (condensed)

**user** — Proceed (with W0.3/W0.4/W0.5).

**assistant** — Discovered `migrations/versions/` was empty (no baseline); tests use
`create_all`. Added the five models + documents delta to `models.py` and authored a four-step
migration chain (baseline + three plan tasks). Verified with a real Postgres via testcontainers:
`upgrade head` clean, `alembic check` reports zero drift.

> **Note:** nothing committed — awaiting HITL commit per governance §9. Next task: **W1.1**
> (auth API endpoints: `/auth/login`, `/auth/refresh`, `/auth/me`, `/auth/logout`).
