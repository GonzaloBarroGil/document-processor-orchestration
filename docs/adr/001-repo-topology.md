# ADR 001 — Repository Topology: Governance Hub + Component Repos

**Status:** Accepted
**Date:** 2026-08-14

## Context

The product family spans three independently buildable surfaces (web service, web app, mobile app)
plus a set of cross-cutting SDD artifacts (constitution, spec, contract, ADRs, glossary,
conversations). We need a topology that keeps the single source of truth (D5) authoritative without
coupling the histories of four codebases.

## Decision

One **governance hub** (`document-processor-orchestration`) plus **one repo per component**
(`document-processor`, `document-processor-web`, `document-processor-mobile`). The hub owns
cross-cutting artifacts only; component repos own code, specs, plans, tasks, and CI.

## Details

- No monorepo, no git submodules.
- The hub's OpenAPI contract is the single source of truth; component repos consume it, they never
  re-author it.
- AI agents may read repo state in all four repos; **only HITL commits** (constitution §9).

## Alternatives Considered

| Option | Pros | Cons |
|--------|------|------|
| Monorepo | Atomic cross-repo changes | Heavy for Python + two TS clients; inconsistent with existing separated-repo pattern |
| Git submodules | Single checkout | Contract drift and history coupling |
| **Hub + component repos** | Clean ownership, independent CI, matches precedent | Cross-repo changes need the contract-change gate |

## Consequences

- Cross-repo contract changes are gated by the contract-change CI workflow (ADR 002, plan §7).
- Each component repo carries the shared `.opencode/` SDD toolset (plan task X1.1).
