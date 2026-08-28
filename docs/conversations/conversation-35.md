# Conversation 35 — CI/CD + follow-ups planning

> **Date:** 2026-08-23
> **Source session:** planning the CI/CD layer and the three follow-ups (governance hub)
> **Outcome:** decision-locked plan for basic CI, CLI alignment, Playwright CI, and the contract gate.
> Pending implementation + HITL commit.

## Summary

HITL asked to analyze the three remaining follow-ups (contract-change gate, `docproc-keys`
alignment, Playwright CI). The analysis surfaced a larger gap — **no repo has basic CI** — and
confirmed the contract-change gate was scaffolded but non-functional (broken web-service
schemathesis step, missing `CROSS_REPO_PAT`). This session locked the plan and adopted GitHub Flow.

## Decisions

| ID | Decision | Status |
|----|----------|--------|
| — | Adopt **GitHub Flow** (main + short-lived feature branches → PRs); CI on `push` (main) + `pull_request` | Accepted |
| — | Include **basic CI** in all four repos (was absent) | Accepted |
| — | Web-service contract check = **static comparison** (build `app.openapi()`, assert contract paths/methods ⊆ app); **schemathesis deferred** (needs Postgres+MinIO+worker+auth) | Accepted |
| — | Backend coverage gate = **measure once + two filtered `coverage report --fail-under`** (domain 90 / adapters 70) — honors constitution, no amendment needed | Accepted |
| — | Maestro (mobile) + Playwright (web) E2E run on **every push** | Accepted |
| — | Hub gets a light CI (OpenAPI YAML parse + spectral) | Accepted |
| — | `CROSS_REPO_PAT` documented as a HITL-only GitHub-secret step on the hub | Accepted |
| — | Branching model documented in `docs/ci-cd.md` **before** the CI is implemented | Accepted |

## Deliverables

- `document-processor-orchestration/docs/ci-cd.md` — branching model + CI matrix + coverage + contract gate + PAT.
- `document-processor-orchestration/docs/feasibility.md` §9 — updated to the implemented state.

## Verification

- N/A (documentation/planning). Implementation verified in conversation-36.

## Transcript (condensed)

**user** — Analyze the three follow-ups.

**assistant** — Found basic CI absent everywhere; the contract gate scaffolded-but-broken;
`docproc-keys` has a constant-prefix bug. Proposed a phased plan.

**user** — Include basic CI; give pros/cons for the contract check; explain CROSS_REPO_PAT.

**assistant** — Recommended static comparison over schemathesis; explained the PAT as a hub
Actions secret.

**user** — GitHub Flow; take the measure-once coverage approach (amend if misaligned).

**assistant** — Measure-once is *more* faithful to the constitution (no amendment needed). Locked the plan.
