# Conversation 8 — Constitution consolidation + docstring enforcement

> **Date:** 2026-08-17
> **Source session:** W0.1 (web-service auth domain) follow-up — constitution consolidation + docstring sweep
> **Outcome:** family constitution v1.0 → v1.1 (backend quality points folded in); backend
> component-level constitution removed; `ruff D` (pydocstyle) enabled and the `src/` docstring
> sweep completed — pending HITL commit.

## Summary

During W0.1 (JWT auth + RBAC foundation in `document-processor`), HITL asked whether the new
`User`/`UserRole`/`AuthPort` classes needed docstrings. That surfaced a gap: **both** constitutions
already mandate Google-style docstrings (§13), but nothing enforced the rule — the existing `src/`
had ~50 public classes/functions with zero docstrings. The backend constitution was also ~90%
redundant with the family constitution. HITL chose to add the docstrings and consolidate the
constitutions, enforcing the rule with `ruff D` and a full sweep rather than grandfathering.

## Decisions

| ID | Decision | Status |
|----|----------|--------|
| — | W0.1 docstrings: `UserRole`, `User`, `AuthPort` get Google-style docstrings (family §13 compliance; no amendment needed — rule already exists) | Applied |
| — | Consolidate: remove backend `docs/constitution.md`; migrate only the 3 backend-only quality points into the family constitution (ruff ruleset, `ruff format`, mypy untyped-lib exemption) | Applied |
| — | Enforce docstrings via `ruff D` (pydocstyle, Google convention) scoped to `src/` (`tests/**` excluded; `D100`/`D104`/`D107` ignored) | Applied |
| — | Sweep existing `src/` (not grandfather): add docstrings to all public classes/functions | Applied |
| — | Retire backend `constitution-drafter` + `constitution.md` command; repoint `.opencode/` at the hub constitution/glossary | Applied |
| — | Archive backend `docs/glossary.md` (hub glossary is the SSoT) | Applied |

## Deliverables

- `document-processor-orchestration/docs/constitution.md` — v1.0 → v1.1: backend static row expanded
  to `ruff (E, F, I, N, W, UP, B, SIM, C4, D) + ruff format + mypy strict`; §7.2 adds "no warnings
  allowed" and the third-party-untyped-library exemption (ADR-011).
- `document-processor/pyproject.toml` — `D` added to `[tool.ruff.lint]`, `convention = "google"`,
  `D100`/`D104`/`D107` ignored, `tests/**` per-file-ignored.
- `document-processor/src/**` — Google-style docstrings added across ~45 files (126 → 0 `D` violations).
- `document-processor/docs/constitution.md` — replaced with a pointer to the hub constitution.
- `document-processor/docs/glossary.md` — replaced with a pointer to the hub glossary.
- `document-processor/.opencode/` — `constitution-drafter` (agent) + `constitution.md` (command)
  removed; remaining agents/commands repointed at the hub constitution/glossary.
- `document-processor-orchestration/docs/feasibility.md` §8.3 — roster + note updated for the retirement.

## Verification

- `ruff check src/ tests/` — clean (0 violations).
- `ruff format --check src/` — clean (12 files reformatted as part of the sweep).
- `pytest tests/unit/` — 64 passed.
- `mypy -p document_processor` — 14 pre-existing errors (ADR-011 untyped-lib exemptions +
  existing `type: ignore` noise), unchanged by this work; no new errors introduced.

## Transcript (condensed)

**user** — I think python docstrings are a good idea so I prefer to add them and make any necessary
constitution amendment.

**assistant** — The requirement already exists (both constitutions §13 mandate Google-style
docstrings), so adding them is compliance, not a new rule. The real gap is *enforcement*. Proposed:
add W0.1 docstrings, and optionally enable `ruff D` + sweep existing `src/`.

**user** — (1) A small dedicated task after W0.1; (2) Ruff D as part of B, sweep existing `src/`;
(3) migrate only necessary points.

**assistant** — Locked in Task A (W0.1 docstrings) + Task B (constitution consolidation +
docstring enforcement). B2 (the sweep) is the bulk; delegated to a subagent.

**assistant** — Sweep complete: 126 → 0 `D` violations, `ruff check src/` passes, mypy unchanged
(14 pre-existing errors, no new).

> **Recovery note:** this session was later recovered (`ses_ffa`); the remaining B2 formatting, B4
> (`.opencode/` retirement + repoint), B5 (glossary archive), and B6 (feasibility §8.3 + this record)
> were completed on resume. Only HITL commits (governance §9).
