# Conversation 39 — Operations guide + backend run/deploy blockers

> **Date:** 2026-08-28
> **Source session:** writing the run/deploy/DB/storage guide and fixing the blockers that made the
> service unrunnable
> **Outcome:** the service now has a production entry point, env template, CORS, and seed tooling;
> `docs/operations.md` documents local dev, deployment, and DB/storage management. Pending HITL commit.

## Summary

HITL asked for a guide on running locally, deploying to production, and managing the database and
storage. Researching the backend surfaced that several things were aspirational or broken — most
importantly, `docker-compose.yml` ran `uvicorn ...main:app` but `main.py` had **no `app` object**
(only the `create_app` factory). Fixed those blockers, then wrote the guide.

## Decisions

| ID | Decision | Status |
|----|----------|--------|
| — | Add a production entry point (`adapters/web/server.py` wiring real adapters + `app`) and point docker-compose at it | Applied |
| — | Commit `.env.example`; document `cp .env.example .env` (the old `cp .env .env.local` was broken) | Applied |
| — | Add CORS middleware to `create_app`, allow-list via `CORS_ALLOW_ORIGINS` | Applied |
| — | Add `docproc-user create <username> <role> <password>` (JWT user seeding) | Applied |
| — | Wire `docproc-lifecycle` (storage sweep) and drop the broken `docproc` script entry | Applied |
| — | Fix `docproc-keys` + `docproc-lifecycle` to `commit()` (they flushed but never committed) | Applied |
| — | Fix `Dockerfile` (COPY `src/` before `pip install .`) + explicit `setuptools` src-layout | Applied |
| — | Document migrations (`alembic upgrade head`; `make migrate`) | Applied |
| — | Guide lives at hub `docs/operations.md`, cross-linked from hub README + `feasibility.md` §11.1 | Applied |

## Deliverables

- **`document-processor`:** `adapters/web/server.py` (new), `core/config.py` (+CORS),
  `adapters/web/main.py` (+CORS), `cli/user.py` (new), `cli/api_keys.py` (+commit),
  `cli/lifecycle.py` (+commit), `pyproject.toml` (scripts + setuptools), `docker-compose.yml`,
  `Dockerfile`, `Makefile` (+migrate), `.env.example` (new), `README.md` (quick start + pointer).
- **`document-processor-orchestration`:** `docs/operations.md` (new), `README.md` (docs index),
  `docs/feasibility.md` §11.1 (pointer), `docs/conversations/conversation-39.md`.

## Verification

- Backend: `ruff check` clean, `ruff format --check` clean, `mypy strict` clean (98 files),
  `pytest tests/unit tests/bdd` → **179 passed**, `scripts/check_contract.py` → 16 ops OK.
- `server.py` and the CLI are statically validated (not run, since they connect to Postgres/MinIO).

## Notes / follow-ups

- `MinioStorage.usage_pct()` computes usage against a **hard-coded 10 GiB quota** — noted in the
  guide as a known limitation (parameterize later).
- The compose `app`/`worker` services and Caddy/prod-compose in `docs/operations.md` are **templates**;
  a real `Caddyfile` and `docker-compose.prod.yml` are not committed (left to the deploy step).
- The existing single-session-per-app wiring (services hold one shared `AsyncSession`) is a pre-existing
  concurrency limitation, out of scope here.

## Transcript (condensed)

**user** — Write a guide on running locally, production deploy, and managing DB & storage.

**assistant** — Researched all four repos; found the API had no `app` entry point, no `.env.example`,
no CORS, no user-seeding, and no migration step. Proposed fixing blockers + writing the guide.

**user** — Fix blockers + write guide; put it at hub `docs/operations.md`.

**assistant** — Implemented the fixes, wrote `docs/operations.md`, cross-linked it, and validated
(ruff/mypy/179 tests/contract check all green).
