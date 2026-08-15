# Conversation 7 — X stream: scaffolding + CI gate + glossary

> **Date:** 2026-08-15
> **Source session:** X stream (governance hub) — continues the SDD pipeline after the Plan phase
> **Outcome:** X1.1 (component `.opencode/` scaffolding), X1.2 (dormant contract-change CI
> workflows), and X1.3 (`docs/glossary.md`) scaffolded — pending HITL commit.

## Summary

Recovered after an unexpected shutdown and executed the **X stream** (governance), which gates
the downstream component-repo SDD phases. Locked decisions from the prior session: (1) scaffold
the two new component repos with a **TS-adapted** `.opencode/` agent/command set while leaving
`document-processor` untouched; (2) create **dormant** GitHub Actions workflows now, to activate
once the repos are pushed. The four local repos now all have their GitHub `origin` remotes.

## Decisions

| ID | Decision | Status |
|----|----------|--------|
| — | `document-processor-web` / `-mobile`: full agent+command set, TS-flavored (`tsc`/ESLint/Vitest/pnpm, Playwright-Maestro), generated-client only | Applied |
| — | `document-processor`: left as-is (no `contract-designer`/`e2e-engineer` added) | Applied |
| — | X1.2 CI gate: dormant workflows in hub + 3 component repos, `repository_dispatch` with a `CROSS_REPO_PAT` secret | Applied |
| — | X1.3 glossary: promote backend terms + new auth/review/export/cost terms | Applied |

## Deliverables

- `docs/glossary.md` (hub) — promoted `document-processor` terms + auth (JWT/refresh/RBAC/API key),
  review (review queue, manual-fix flag), export, cost-governance, offline-queue terms.
- `document-processor-web/.opencode/` — 5 agents + 5 commands, TS-flavored.
- `document-processor-mobile/.opencode/` — 5 agents + 5 commands, TS-flavored.
- `.github/workflows/contract-change.yml` (hub) — dispatches contract PRs to the 3 consumers.
- `.github/workflows/contract-check.yml` (web, mobile, service) — regenerate client / schemathesis
  against the hub contract.
- `README.md` indexed `docs/glossary.md`; `feasibility.md` §13 steps 3 & 5 marked scaffolded.

## Transcript (condensed)

**user** — Another unexpected shutdown. Please recover our last session. No changes in any working
tree of the four repos. We made some decisions on scaffolding. I also added the remotes for the four
local repos.

**assistant** — Recovered `ses_ffde` (crashed mid X-stream after the two forks were answered).
Verified all four working trees clean and `origin` remotes present on each repo. Executed X1.3
(glossary), X1.1 (TS-adapted `.opencode/` scaffolding for web + mobile), and X1.2 (dormant CI
workflows in hub + 3 component repos). Per governance §8.4, no commit — this record persists the
session (D10).

---

## Extension — component `constitution-drafter` retired

> **Date:** 2026-08-15 (same session)

HITL flagged that a `constitution-drafter` in the component repos was inconsistent with the family
constitution (§2, §8): the Constitution is a hub-owned artifact, and component repos conform to it,
entering the SDD pipeline at **Spec**. Corrected as follows:

- Removed `.opencode/agents/constitution-drafter.md` and `.opencode/commands/constitution.md` from
  `document-processor-web` and `document-processor-mobile`.
- Pointed the remaining agents/commands at the family constitution
  (`document-processor-orchestration/docs/constitution.md`) instead of a local `@docs/constitution.md`.
- `document-processor` left as-is; its `constitution-drafter` is historical and open to future
  refactor as the backend evolves under the orchestration model.
- Amended `docs/constitution.md` §8 (component repos enter at Spec), `docs/feasibility.md` §8.3
  (roster), and `docs/plan.md` §7 (X1.1 wording) for consistency.

**Decisions:**

| ID | Decision | Status |
|----|----------|--------|
| — | No per-component `constitution-drafter`; Constitution is hub-owned, components enter at Spec | Applied |
| — | Backend `constitution-drafter` kept for now, open to future refactor | Applied |
