# Conversation 9 — W0.2 auth services + JWT secret hardening

> **Date:** 2026-08-19
> **Source session:** W0.2 (JWT + argon2id auth service layer) in `document-processor`
> **Outcome:** token/password/auth services implemented and green; mypy 14 → 0; `jwt_secret`
> hardened with a 32-byte minimum. Pending HITL commit.

## Summary

Resumed the W-stream after W0.1 (auth domain). Closed the two open items from the prior session
(the 14 pre-existing mypy errors and the trailing NUL bytes in `feasibility.md`), then delivered
W0.2: JWT issuance/verification/rotation, argon2id password hashing, and the `AuthService`
orchestration. HITL chose `PyJWT` over `python-jose` and a dedicated `TokenService` (defaults
offered in the plan). During verification, an `InsecureKeyLengthWarning` surfaced a 20-byte dev
`jwt_secret`; fixed by lengthening the default and enforcing `min_length=32`.

## Decisions

| ID | Decision | Status |
|----|----------|--------|
| — | W0.2 JWT library: `PyJWT` (minimal, maintained, OSI) over `python-jose` | Applied |
| — | Separate `TokenService` (pure JWT) + `PasswordHasher` (argon2id) + `AuthService` (orchestration), not one monolithic service | Applied |
| — | `jwt_secret` strict `Field(min_length=32)` + ≥32-byte dev default (fixes `InsecureKeyLengthWarning`; fails fast on short env secret) | Applied |
| — | Refresh rotation: decode → `is_active` check → revoke → issue new pair (rotation safety in `AuthService`, not `TokenService`) | Applied |
| — | **Deferred:** `StarletteDeprecationWarning` in `document-processor` test stack (`tests/bdd/conftest.py:5`, `starlette.testclient` + `httpx`; "install httpx2"). Upstream dependency drift — swap the test HTTP client when addressed. | Deferred |
| — | **Deferred:** `document-processor/tests/` not `ruff format`-ed (pre-existing gap; only `src/` was formatted). Left untouched to keep the W0.2 diff focused. | Deferred |

## Deliverables

- `document-processor/pyproject.toml` — `pyjwt>=2.9`, `argon2-cffi>=23.1` added to deps.
- `document-processor/src/document_processor/core/config.py` — `jwt_secret` (32-byte min),
  `jwt_algorithm`, `access_token_ttl_seconds`, `refresh_token_ttl_seconds`.
- `document-processor/src/document_processor/core/errors.py` — `InvalidCredentialsError`,
  `InvalidTokenError`.
- `document-processor/src/document_processor/domain/models/token.py` — `TokenType`,
  `TokenPayload`, `RefreshToken`, `TokenPair`.
- `document-processor/src/document_processor/domain/ports/refresh_token.py` —
  `RefreshTokenRepositoryPort` (save / is_active / revoke).
- `document-processor/src/document_processor/domain/services/password_hasher.py` — argon2id
  hash/verify.
- `document-processor/src/document_processor/domain/services/token_service.py` — JWT
  issue/decode/rotate + `hash_token` (SHA-256).
- `document-processor/src/document_processor/domain/services/auth_service.py` — `authenticate`
  + `refresh` (rotating refresh).
- `document-processor/tests/unit/domain/` — `test_token_service.py`, `test_password_hasher.py`,
  `test_auth_service.py` (17 new tests).
- `document-processor-orchestration/docs/feasibility.md` — 3,049 trailing NUL bytes stripped.
- `document-processor` mypy cleanups (10 files): ADR-011 import exemptions, missing annotations,
  stale `type: ignore`, `CursorResult` cast, `EntryPoints` fallback removed.

## Verification

- `mypy -p document_processor` — 0 errors (was 14 pre-existing; 71 source files).
- `ruff check src/ tests/` — clean.
- `ruff format --check src/` — clean.
- `pytest tests/unit tests/bdd` — 116 passed, 1 warning (the deferred Starlette one).

## Transcript (condensed)

**user** — Proceed with W0.2 (defaults: PyJWT + separate TokenService).

**assistant** — Implemented token/password/auth services with TDD; 17 new unit tests pass; mypy/ruff green.

**user** — Are those warnings tech debt? Fix now?

**assistant** — Two warning groups: 11× `InsecureKeyLengthWarning` (new — 20-byte dev `jwt_secret`,
security smell) and 1× `StarletteDeprecationWarning` (pre-existing test-stack dep). Fix #1, defer #2.

**user** — Fix #1 now; record #2 in the conversation transcript; keep `min_length` strict.

**assistant** — Applied the `jwt_secret` hardening; warnings 27 → 1 (deferred Starlette only). This
record captures both deferred items.
