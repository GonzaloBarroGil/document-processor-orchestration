# Conversation 40 — Follow-ups: storage quota, deploy artifacts, request-scoped sessions

> **Date:** 2026-08-31
> **Source session:** resolving the three follow-ups logged in conversation-39
> **Outcome:** storage quota parameterized, real production deploy artifacts committed, and the
> backend's shared-session concurrency + missing-commit bug fixed with request-scoped sessions.
> Pending HITL commit.

## Summary

Conversation-39 closed with three follow-ups: (1) `MinioStorage.usage_pct()` divided by a hard-coded
10 GiB quota, (2) the Caddyfile / `docker-compose.prod.yml` existed only as inline docs, and (3) the
production entry point shared one `AsyncSession` across all requests. During planning, (3) turned out
to be **two** bugs — a concurrency hazard *and* a missing commit boundary (every repository only
`flush()`es, so ingested documents/users/audit would never persist under `server:app`). The worker had
the same class of bug. All three are now fixed.

## Decisions

| ID | Decision | Status |
|----|----------|--------|
| F1 | Parameterize the storage quota as `STORAGE_QUOTA_BYTES` (default 10 GiB) | Applied |
| F2 | Commit `deploy/Caddyfile` + `deploy/docker-compose.prod.yml` + `deploy/.env.production.example` as templates (placeholder domains) | Applied |
| F3 | One `AsyncSession` per request via FastAPI dependency injection (commit/rollback per request); worker commits per document | Applied |
| F3a | Auth middleware validates keys on its own short-lived session (`SessionScopedApiKeyRepository`) | Applied |
| F3b | `create_app` gains `include_all_routers=True`; keeps the built-services path for tests | Applied |
| F3c | Worker `DocumentService` rebuilt per document in a fresh, committed session | Applied |

## Deliverables

- **`document-processor`:** `core/config.py` (+`storage_quota_bytes`), `adapters/storage/minio.py`
  (use setting), `.env.example` (+`STORAGE_QUOTA_BYTES`), `.gitignore` (+`.env.production`),
  `adapters/web/main.py` (optional services + `include_all_routers`), `adapters/web/server.py`
  (request-scoped wiring + `ServiceContext` + `SessionScopedApiKeyRepository` + dependency overrides),
  `cli/worker.py` (per-document session + commit), `deploy/Caddyfile` (new),
  `deploy/docker-compose.prod.yml` (new), `deploy/.env.production.example` (new),
  `tests/unit/adapters/test_storage.py` (+2 usage_pct tests), `docs/adr/012-request-scoped-sessions.md`
  (new), `README.md` (12 ADRs).
- **`document-processor-orchestration`:** `docs/operations.md` (env table + quota note; §4.3/§4.4 now
  reference the committed deploy files; §4.5–§4.7 use `-f deploy/docker-compose.prod.yml`),
  `docs/conversations/conversation-40.md`.

## Verification

- Backend: `ruff check` clean, `mypy -p document_processor` clean (98 files),
  `pytest tests/unit tests/bdd` → **181 passed**, `pytest tests/integration` → **54 passed**,
  `pytest tests/contract` → **15 passed**, `scripts/check_contract.py` → 16 ops OK.
- `docker compose -f deploy/docker-compose.prod.yml config --quiet` → OK.
- `server.py`/worker/CLI validated statically (not run — they connect to Postgres/MinIO).

## Notes / follow-ups

- `ruff format --check` reports ~12 pre-existing files "would reformat" (a local ruff-version
  difference vs. CI); none are touched by this session's changes, whose files are all format-clean.
- The API's `DocumentService` path never runs OCR (ingest-only), so the shared OCR engine in the
  request-scoped context is safe; the worker builds its own.
- `MinioStorage` methods are `async def` but call the sync `minio` client (blocks the event loop) — a
  pre-existing concern, out of scope here.

## Transcript (condensed)

**user** — All changes committed.

**user** — What's the plan for the follow-ups?

**assistant** — Sized the three follow-ups; found (3) is two bugs (shared session + no commit).
Proposed F1/F2/F3 with approach A (idiomatic per-request session) and placeholder-domain deploy files.

**user** — (F1 + F2 + F3; approach A; commit deploy files as templates; split into subtasks with
consistent commits and keep docs/source-of-truth updated.)

**user** — Proceed.

**assistant** — Implemented F1 (quota), F2 (deploy artifacts), F3 (request-scoped sessions + worker
commit fix), ADR-012, and this transcript; verified ruff/mypy/250 tests/contract/compose config.
