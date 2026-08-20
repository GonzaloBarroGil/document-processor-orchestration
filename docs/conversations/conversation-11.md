# Conversation 11 — W1.1 auth API endpoints

> **Date:** 2026-08-19
> **Source session:** W1.1 (auth API endpoints) in `document-processor`
> **Outcome:** `/auth/login`, `/auth/refresh`, `/auth/me`, `/auth/logout` implemented, wired, and
> verified end-to-end against Postgres 16. Pending HITL commit.

## Summary

Implemented the four auth endpoints on top of W0.2 (services) and W0.3 (migrations). Added the
`AuthPort` and `RefreshTokenRepositoryPort` adapter implementations, a `get_current_user` Bearer
dependency, and updated the API-key middleware to let the login/refresh endpoints through
unauthenticated and defer the Bearer-only endpoints to the router.

## Decisions

| ID | Decision | Status |
|----|----------|--------|
| — | `AuthService` gains `current_user(access_token)` and `logout(refresh_token)` (logout revokes the supplied token; no-op if absent) | Applied |
| — | `create_app` gains an **optional** `auth_service` param (default `None`); the auth router is registered only when provided, so existing BDD/integration fixtures are untouched | Applied |
| — | Middleware: `/auth/login` + `/auth/refresh` are public; `/auth/me` + `/auth/logout` are Bearer-passthrough (validated by `get_current_user`, not the API-key middleware) | Applied |
| — | `logout` accepts an **optional** `refresh_token` body (superset of the contract, which omits a body) to honor ADR-003's "revokes the current refresh token" | Applied |

## Deliverables

- `document-processor/src/document_processor/domain/services/auth_service.py` — `current_user`,
  `logout`.
- `document-processor/src/document_processor/adapters/persistence/postgresql/user_repository.py` —
  `PostgresUserRepository` (`AuthPort`).
- `document-processor/src/document_processor/adapters/persistence/postgresql/refresh_token_repository.py` —
  `PostgresRefreshTokenRepository` (`RefreshTokenRepositoryPort`).
- `document-processor/src/document_processor/adapters/persistence/postgresql/models.py` —
  `user_to_model` / `model_to_user`.
- `document-processor/src/document_processor/adapters/web/api/auth.py` — 4 endpoints + request schemas.
- `document-processor/src/document_processor/adapters/web/api/deps.py` — `set/get_auth_service`,
  `get_current_user`.
- `document-processor/src/document_processor/adapters/web/main.py` — optional `auth_service` wiring.
- `document-processor/src/document_processor/adapters/web/middleware/auth.py` — public/bearer paths.
- `document-processor/tests/unit/adapters/test_auth_repositories.py`,
  `tests/integration/test_auth_api.py`, and `tests/unit/domain/test_auth_service.py` (extended).

## Verification

- `mypy -p document_processor` — clean (78 files).
- `ruff check src tests` — clean; `ruff format --check src` — clean.
- `pytest tests/unit tests/bdd` — 129 passed; `pytest tests/integration` — 27 passed.
- Full flow against Postgres 16 (testcontainers): create user → login → current_user → refresh
  (rotation) → logout — all pass.

## Transcript (condensed)

**user** — Proceed (with W1.1).

**assistant** — Implemented the four auth endpoints, the `AuthPort`/`RefreshTokenRepositoryPort`
adapters, and the Bearer dependency; updated the middleware and `create_app`. Verified statically
and end-to-end against real Postgres.

> **Note:** nothing committed by the agent — awaiting HITL commit per governance §9. Next task:
> **W1.2** (RBAC dependency `ADMIN`/`REVIEWER`).
