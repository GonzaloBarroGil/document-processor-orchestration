# Conversation 12 — W1.2 RBAC dependencies

> **Date:** 2026-08-19
> **Source session:** W1.2 (RBAC dependency) in `document-processor`
> **Outcome:** `require_roles` / `require_admin` / `require_reviewer` FastAPI guards added and tested.
> Pending HITL commit.

## Summary

Added the role-based access control dependencies on top of the W1.1 `get_current_user` Bearer
dependency. `require_roles(*roles)` returns a `Depends` guard that rejects with 403 when the
current user's role is not in the allowed set; `require_admin` and `require_reviewer` are
module-level singletons. The guards are not yet attached to any route — that happens in W1.4
(review endpoints) and the API-key admin UI (F14).

## Decisions

| ID | Decision | Status |
|----|----------|--------|
| — | Set-based guard `require_roles(*roles)` (explicit allow-list) rather than an ordinal role hierarchy | Applied |
| — | `require_admin = require_roles(ADMIN)`; `require_reviewer = require_roles(ADMIN, REVIEWER)` (ADMIN ⊃ REVIEWER per ADR-003) | Applied |
| — | 403 `Insufficient role` returned as `HTTPException` (matches contract `Forbidden` shape) | Applied |

## Deliverables

- `document-processor/src/document_processor/adapters/web/api/deps.py` — `require_roles`,
  `require_admin`, `require_reviewer`.
- `document-processor/tests/unit/adapters/test_rbac_deps.py` — guard allow/reject logic.
- `document-processor/tests/integration/test_rbac.py` — end-to-end `get_current_user →
  require_admin` chain (ADMIN 200, REVIEWER 403).

## Verification

- `mypy -p document_processor` — clean (78 files).
- `ruff check src tests` — clean; `ruff format` — clean.
- `pytest tests/unit tests/bdd` — 135 passed; `pytest tests/integration` — 29 passed.

## Transcript (condensed)

**user** — Proceed to W1.2.

**assistant** — Added `require_roles`/`require_admin`/`require_reviewer` to `deps.py` with unit and
end-to-end tests. Note: the guards are inert until W1.4 wires them to review routes, and the
middleware `BEARER_PATHS` set will need the review routes added (or a prefix-based refactor).

> **Note:** nothing committed by the agent — awaiting HITL commit per governance §9. Next task:
> **W1.3** (review service) then **W1.4** (review endpoints + RBAC wiring).
