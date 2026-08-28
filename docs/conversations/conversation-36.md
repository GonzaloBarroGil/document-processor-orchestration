# Conversation 36 — CI/CD + follow-ups implementation

> **Date:** 2026-08-23
> **Source session:** implementing the CI/CD plan (all four repos)
> **Outcome:** basic CI everywhere, CLI aligned, static contract gate. Pending HITL commit.

## Summary

Executed the plan from conversation-35: added basic CI to all four repos (GitHub Flow triggers),
aligned the `docproc-keys` CLI on `ApiKeyService`, and replaced the broken schemathesis step with a
static contract comparison. Documented everything in `docs/ci-cd.md`.

## Decisions

| ID | Decision | Status |
|----|----------|--------|
| — | CI triggers: `push` (main) + `pull_request` (GitHub Flow) | Applied |
| — | Backend coverage: measure once (`pytest --cov=src --cov-report=`) then two filtered `coverage report --include --fail-under` (domain 90 / adapters 70) — validated locally (99% / 72%) | Applied |
| — | Web-service contract check: `scripts/check_contract.py` (build `app.openapi()`, assert contract operations ⊆ app) + spectral; schemathesis dropped | Applied |
| — | `pip install -e ".[dev]"` for the service CI (no `uv.lock` exists); added `pyyaml` to dev deps | Applied |
| — | Maestro (mobile) + Playwright (web) E2E moved to every push | Applied |
| — | `CROSS_REPO_PAT` remains a HITL-only GitHub-secret step (documented in `docs/ci-cd.md` §5) | Applied |

## Deliverables

- Hub: `docs/ci-cd.md`, `docs/conversations/conversation-35.md`, `docs/feasibility.md` §9 update,
  `.github/workflows/ci.yml` (YAML parse + spectral).
- `document-processor`: `.github/workflows/ci.yml`, `scripts/check_contract.py`,
  `.github/workflows/contract-check.yml` (static comparison), `pyproject.toml` (+pyyaml),
  `cli/api_keys.py` (on `ApiKeyService`).
- `document-processor-web`: `.github/workflows/ci.yml` (lint/typecheck/test/build + Playwright).
- `document-processor-mobile`: `.github/workflows/ci.yml` (…+ coverage gate), `mobile-e2e.yml`
  (push/PR trigger).

## Verification

- `document-processor`: ruff/mypy/format clean; 179 unit/BDD + integration tests pass;
  `check_contract.py` → "OK: all 16 contract operations implemented."
- Coverage gates validated locally: domain 99%, adapters 72%.
- Web/mobile workflows reference existing `pnpm` scripts (unchanged code).

## Notes / follow-ups

- **Spectral + hub CI can't be run locally** (no Node spectral); the first CI run may reveal the
  default `oas` ruleset flags 3.1 constructs — if so, add a `.spectral.yaml` ruleset or fix the
  contract.
- **`CROSS_REPO_PAT`** still needs to be created (HITL, hub → Settings → Secrets → Actions) before
  the `repository_dispatch` gate actually fires.
- Schemathesis remains deferred (needs Postgres + MinIO + worker + seeded auth).

## Transcript (condensed)

**user** — Proceed.

**assistant** — Implemented basic CI across the four repos, aligned the CLI, and added the static
contract gate; verified locally (coverage gates + check_contract.py green).
