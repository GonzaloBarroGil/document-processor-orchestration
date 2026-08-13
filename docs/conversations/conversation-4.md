# Conversation 4 — Spec phase: OpenAPI contract (draft)

> **Date:** 2026-08-13
> **Source session:** continuation of `ses_008a8fee4ffeTqanPcqXgatHFi` (Spec phase)
> **Outcome:** drafted `docs/contracts/openapi.yaml` v1.0 — **approved by HITL**.

## Summary

Entered the **Spec** phase. Drafted the unified OpenAPI 3.1 contract — the single source of
truth (D5). It promotes the existing `document-processor` `/api/v1` endpoints unchanged
(API-stability rule) and adds the family endpoints: JWT auth (D9), manual review + export for
the web app, and the D12 cost-control guards (rate limit + daily quota documented on ingest).

## Decisions

| ID | Decision | Status |
|----|----------|--------|
| — | Promote `/api/v1` endpoints as-is; add auth/review/export as additive paths (no breaking change) | Applied |
| — | Two security schemes: `X-API-Key` (machine) + JWT Bearer (human), `OR` on shared operations | Applied |

## Transcript (condensed)

**user** — Constitution approved. Changes staged but not committed; will you modify status before commit?

**assistant** — Marked `docs/constitution.md` as "Approved v1.0" and updated references.

**user** — Committed and pushed.

**assistant** — Started Spec phase: read `document-processor` routes (`documents.py`, `health.py`),
domain models (`document.py`, `parsed_data.py`, `validation.py`, `api_key.py`), CLI, and
`docs/spec.md` (9 features / 31 scenarios). Drafted `docs/contracts/openapi.yaml` (11 paths)
covering health, auth (login/refresh/me/logout), documents (ingest/list/get/image), review
(review/queue/export), with schemas matching the existing domain model plus `user_id` (D9) and
D12 guards. Updated README and feasibility §13.
