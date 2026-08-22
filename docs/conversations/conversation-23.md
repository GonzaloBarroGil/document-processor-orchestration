# Conversation 23 — F14 API key administration (contract + backend + A1.6 UI)

> **Date:** 2026-08-21
> **Source session:** F14 (API key admin) across the three repos
> **Outcome:** contract, backend endpoints, and web UI for machine-client API key management.
> Pending HITL commit.

## Summary

Closed the F14 gap (the contract/backend previously had no API-key admin surface). Added the
contract paths + schemas, backend endpoints (ADMIN, Bearer) backed by an `ApiKeyService`, and the
A1.6 web UI. Preceded by the "either credentials" blocker fix so the web app (Bearer) can reach the
documents/export endpoints.

## Decisions

| ID | Decision | Status |
|----|----------|--------|
| — | Blocker fix: `AuthMiddleware` passes `Bearer` requests through for documents paths; `get_optional_current_user` (nullable auth service) validates the Bearer on ingest/list/detail/image/export | Applied |
| — | API keys: `POST/GET /api/v1/api-keys` + `POST /api/v1/api-keys/{prefix}/revoke`, `require_admin`, Bearer; `/api/v1/api-keys` treated Bearer-only in middleware | Applied |
| — | Key generation: raw `sk-proj-<token>`, **unique** prefix `token[:8]` (the pre-existing CLI used the constant `sk-proj-` prefix — left untouched, flagged for a follow-up) | Applied |
| — | `ApiKeyService` owns issuance; `ApiKeyRepositoryPort` extended with `create`/`list_keys`/`revoke` | Applied |
| — | Raw key returned once in the create response; list excludes the hash (`ApiKeyView`) | Applied |

## Deliverables

- `document-processor-orchestration/docs/contracts/openapi.yaml` — api-keys paths + 4 schemas.
- `document-processor` — `ApiKeyService`, `ApiKeyNotFoundError`, `api/api_keys.py` router,
  `ApiKeyRepositoryPort`/`PostgresApiKeyRepository` methods, `ApiKey` model (+`label`),
  `CreatedApiKey`, `get_optional_current_user` + middleware/middleware bearer handling.
- `document-processor-web` — regenerated client; `features/api-keys/` (list/create/revoke UI);
  `/api-keys` route wired to the real view.

## Verification

- Backend: mypy/ruff clean; `177` unit/BDD + `52` integration tests.
- Web: typecheck/lint clean; `47` tests; build succeeds.
- Contract: YAML parses (13 paths, 23 schemas).

## Transcript (condensed)

**user** — Proceed with F14.

**assistant** — Fixed the "either credentials" blocker, then added the API-key admin contract +
backend + web UI end-to-end; verified green.

> **Note:** nothing committed by the agent — awaiting HITL commit per governance §9. Next: **F15**
> (dashboard stats endpoint + A1.7 UI). Follow-up: align `docproc-keys` CLI on `ApiKeyService`.
