# CI/CD — Continuous Integration & Contract Gate

> **Date:** 2026-08-23
> **Status:** Adopted
> **Scope:** all four repos — `document-processor-orchestration` (hub), `document-processor`
> (web service), `document-processor-web`, `document-processor-mobile`.

This document defines the branching model and the CI matrix. It is the operational complement to
`docs/constitution.md` §7 (Quality Standards) and §9 (Git Governance), and supersedes the
aspirational `docs/feasibility.md` §9.

---

## 1. Branching model — GitHub Flow

- **`main`** is always deployable. No long-lived branches.
- Every change ships on a **short-lived feature/fix branch** and is merged via a **pull request**.
- CI runs on **every push to `main`** and on **every `pull_request`** (the PR branch), so a change
  is verified before merge.

This is the lightest model that still gives the "CI green before merge" gate. It also aligns with
the contract-change gate (§4), which is triggered by a PR touching `docs/contracts/**`.

---

## 2. Per-component CI matrix

All workflows trigger on `push` (to `main`) and `pull_request`.

| Repo | Steps (in order) | Coverage gate |
|------|------------------|---------------|
| `document-processor` (W) | `pip install -e ".[dev]"` → `ruff check src/ tests/` → `ruff format --check src/` → `mypy -p document_processor` → `pytest tests/unit tests/bdd` → `pytest tests/integration` (testcontainers: Postgres + MinIO) | domain ≥90%, adapters ≥70% (measure-once, two filtered `coverage report` passes — see §3) |
| `document-processor-web` (A) | `pnpm install --frozen-lockfile` → `lint` → `typecheck` → `test -- --run` → `build` | — (no numeric gate yet) |
| `document-processor-mobile` (M) | `pnpm install --frozen-lockfile` → `lint` → `typecheck` → `test -- --run --coverage` → `build` | ≥80% (configured in `vite.config.ts`) |
| `document-processor-orchestration` (X) | Python YAML parse of `docs/contracts/openapi.yaml` → `spectral` lint | — |

### E2E (on every push)

- **Web app** — Playwright (`e2e/app.spec.ts`), self-contained via `page.route` API mocks.
  Browser installed with `pnpm exec playwright install --with-deps chromium`.
- **Mobile app** — Maestro (`e2e/maestro/*.yaml`) on an Android emulator; workflow builds the
  web bundle → `cap add/sync android` → `gradle assembleDebug` → `maestro test`.

---

## 3. Backend coverage gate (measure once, two filtered passes)

The constitution requires **domain ≥90%** and **adapters ≥70%**. coverage.py cannot express two
per-path thresholds in one report, so the CI measures coverage once and enforces each layer with a
separate filtered report:

```bash
coverage run -m pytest tests/unit tests/bdd          # measure once
coverage report --include="*/domain/*"  --fail-under=90
coverage report --include="*/adapters/*" --fail-under=70
```

---

## 4. Contract-change gate (cross-repo)

The OpenAPI contract (`docs/contracts/openapi.yaml`) is the single source of truth (D5).

1. A PR to the **hub** touching `docs/contracts/**` triggers `contract-change.yml`, which
   `repository_dispatch`es `contract-change` to the three component repos with the PR head SHA.
2. Each consumer's `contract-check.yml` fetches the contract at that SHA and verifies:
   - **web / mobile** — regenerate the TS client, `typecheck`, `test`.
   - **web service** — `spectral` lint + `scripts/check_contract.py` (static: build the FastAPI
     app, `app.openapi()`, assert every contract path/method is implemented).

The dispatch requires a **`CROSS_REPO_PAT`** repository secret on the hub (see §5). Schemathesis
(runtime contract fuzzing against a live server) is a **documented deferred enhancement** — it
needs Postgres + MinIO + a worker + seeded auth and is not yet wired.

Spectral uses a committed `.spectral.yaml` ruleset (`extends: spectral:oas`, with the metadata-only
recommended rules `info-contact`/`operation-description`/`operation-operationId` disabled). It is
present in both the hub and the web service so `spectral lint` auto-discovers it.

---

## 5. `CROSS_REPO_PAT` — setup (HITL-only step)

This is a GitHub Personal Access Token stored as an Actions secret on the **hub repo**. It is not
stored in any file.

1. **Create a fine-grained PAT** (GitHub → Settings → Developer settings → Personal access tokens →
   Fine-grained tokens → Generate new token):
   - **Resource owner:** your account.
   - **Repository access:** "Only select repositories" → `document-processor`,
     `document-processor-web`, `document-processor-mobile` (the three *consumers* the hub dispatches to).
   - **Permissions → Repository permissions → Administration: Read and write** (required to trigger
     `repository_dispatch`). Metadata is auto-selected.
   - (Alternative: a **classic** token with the `repo` scope.)
2. **Store it**: hub repo (`document-processor-orchestration`) → **Settings → Secrets and variables →
   Actions → New repository secret**, name **`CROSS_REPO_PAT`**, paste the token.
3. Only the hub uses it (the `repository_dispatch` call). Consumers fetch the raw contract
   unauthenticated (public repos), so they need no secret.

---

## 6. Follow-ups

- Schemathesis runtime contract check (deferred).
- `docproc-keys` CLI already aligned on `ApiKeyService` (see conversations).
- Hub doc-quality checks (markdown lint, glossary drift) can be added to the hub CI later.
