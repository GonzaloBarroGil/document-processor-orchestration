# ADR 002 — Contract-First with Generated TypeScript Clients

**Status:** Accepted
**Date:** 2026-08-14

## Context

Three components must agree on request/response shapes for every shared operation. Hand-written
DTOs on both sides inevitably drift, and the feasibility analysis (§10) rates contract drift as a
high-impact risk across three repos.

## Decision

The OpenAPI 3.1 contract in `docs/contracts/openapi.yaml` is the **single source of truth**. The web
service *implements* it; the web and mobile apps *consume* it via **generated** TypeScript clients.

## Details

- Client codegen: `openapi-typescript` + `openapi-fetch` (minimal, type-safe; `orval` only if
  TanStack Query per-hook generation is later needed).
- Server schemas are validated at runtime by FastAPI/Pydantic (already true).
- Cross-component DTOs are never hand-written twice.
- A contract change in the hub triggers regeneration + compile in both client repos before merge.

## Alternatives Considered

| Option | Pros | Cons |
|--------|------|------|
| Hand-written DTOs | Simple | Drift risk; violates single-source-of-truth |
| `orval` | React Query hooks out of the box | Heavier; unneeded before data-fetch requirements firm up |
| **openapi-typescript + openapi-fetch** | Minimal, type-safe | No per-hook codegen |

## Consequences

- Both client repos contain a generated `src/api/` directory that is never hand-edited.
- Contract drift surfaces as a compile error, not a runtime bug.
