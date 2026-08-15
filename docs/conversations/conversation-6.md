# Conversation 6 — Plan phase: cross-component plan + ADRs (draft)

> **Date:** 2026-08-14
> **Source session:** Plan phase (continues the governance-hub SDD pipeline)
> **Outcome:** drafted `docs/plan.md` v1.0 + 5 cross-cutting ADRs — **approved by HITL**.

## Summary

Entered the **Plan** phase of the SDD pipeline. Decomposed the approved product spec
(F1–F18) into ordered, testable work across the three component repos plus the governance
hub, and recorded the new cross-cutting decisions as ADRs. Backend features F1–F9 are
already delivered in `document-processor`; the plan therefore focuses on the additive web
service work (auth D9, review/export, audit + dead-letter, daily quota D12) and the two new
apps (web SPA, mobile Capacitor).

## Decisions

| ID | Decision | Status |
|----|----------|--------|
| — | Plan is cross-component, owned by the hub; per-component plans/tasks live in each component repo later | Applied |
| — | Contract-first ordering: web service first → generated clients → web/mobile consume | Applied |
| — | `docs/glossary.md` (missing §13 deliverable) folded in as task X1.3 | Applied |
| ADR 001 | Repository topology (D1) | Accepted |
| ADR 002 | Contract-first + generated TS clients (D5) | Accepted |
| ADR 003 | JWT + rotating refresh + RBAC (D9) | Accepted |
| ADR 004 | Human-in-the-loop review workflow (resolves §11.3 Q1) | Accepted |
| ADR 005 | Public-exposure cost & abuse governance (D12) | Accepted |

## Transcript (condensed)

**user** — Lets proceed with plan phase.

**assistant** — Read the constitution, product spec, and OpenAPI contract, plus the
`document-processor` plan/ADR precedents. Drafted `docs/plan.md` (33 tasks: W=10 web service,
A=11 web app, M=8 mobile, X=3 governance) with data-model deltas and dependency order, and 5
cross-cutting ADRs (`docs/adr/001–005`). Flagged the missing `docs/glossary.md` (§13).

**user** — Changes approved. Please update conversation and status before HITL commit.

**assistant** — Marked `docs/plan.md` "Approved v1.0", indexed `docs/plan.md` + `docs/adr/`
in the README, and wrote this conversation record.
