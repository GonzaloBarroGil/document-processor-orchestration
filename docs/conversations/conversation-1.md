# Conversation 1 — Feasibility analysis + decision log approval

> **Date:** 2026-08-12
> **Source session:** `ses_008a8fee4ffeTqanPcqXgatHFi` (title: *Orchestration repo feasibility analysis with SDD*)
> **Outcome:** `docs/feasibility.md` drafted (v0.1 → v0.2); decision log D1–D10 accepted; Hosting left pending.

## Summary

The session produced the orchestration repo's feasibility analysis and then extended it in a
follow-up focused on Hosting, Auth, and conversation persistence. All recommendations in the
decision log were accepted by the HITL. The only open decision remaining is **Hosting**
(hybrid-managed vs self-host VPS), deferred to the Spec phase.

## Decisions

| ID | Decision | Status |
|----|----------|--------|
| D1 | Governance-hub repo + separate component repos | Accepted |
| D2 | Backend stays Python 3.12 + FastAPI (no TS migration) | Accepted |
| D3 | Web app: React + Vite + TS SPA | Accepted |
| D4 | Mobile app: Capacitor + React (Flutter fallback) | Accepted |
| D5 | Single OpenAPI 3.1 contract owned here + generated TS clients | Accepted |
| D6 | Adopt document-processor SDD pipeline + agents + HITL git governance | Accepted |
| D7 | Reject ICES AWS Textract/BullMQ; keep PaddleOCR/EasyOCR + PG queue | Accepted |
| D8 | Merge ICES audit-trail + failed-extraction ideas into schema | Accepted |
| D9 | Authentication: JWT (web + mobile) with rotating refresh tokens, roles `ADMIN`/`REVIEWER`; API keys retained for machine clients | Accepted |
| D10 | Conversation persistence: every Validation stage / HITL gate persists the session to `docs/conversations/` before passing | Accepted |

## Transcript (condensed)

**user** — Hi, I want this to be the orchestration repo for a combination of web service, web app
and mobile app, taking the work made for `../document-processor/` (and maybe also `../ICES`) as a
starting point, but not necessarily the definitive stack. I want to start with a new feasibility
analysis, focused on a strict SDD environment, high coverage, unit, automation and integration tests.

**assistant** — Reviewed `document-processor`, `ICES`, and the `cesar-orchestration` /
`news-app-feasibility` patterns; asked to lock scope decisions before writing.

**assistant** — Understood: governance-hub repo with separate component repos; stack and mobile
framework evaluated in the doc; deliverable is the feasibility analysis only. Writing it now.

**assistant** — Done. Created `docs/feasibility.md` (423 lines). Key recommendations: governance hub
topology; keep Python/FastAPI; React + Vite SPA; Capacitor + React mobile (Flutter fallback); single
OpenAPI 3.1 contract; per-layer testing matrix; promote document-processor SDD + two new agents.
Decisions D2–D5 and D8 marked "Recommended (HITL confirm)".

**user** — That's a good feasibility analysis. First of all, the decision log: Agreed with all of
them. I want to go further with the open questions, in particular, Hosting & Auth. Additional points:
given my frequent unexpected shutdowns and your skills recovering opencode sessions, I want to enforce
the update of conversations on every validation stage.

**assistant** — Proposed a plan (no edits yet): (A) lock D1–D8 as Accepted; (B) resolve Hosting with a
per-layer recommendation table (Fly.io OCR worker as the decisive factor); (C) resolve Auth with JWT +
roles + retained API keys; (D) enforce conversation persistence at every Validation stage via the
validator agent + `/validate` command, with `resume-session` as recovery.

**assistant** — Final plan (ready to execute on approval): update `docs/feasibility.md` (decision log,
§8 gate, §11 hosting pros/cons + auth resolution); scaffold `.opencode/` (`opencode.json`,
`agents/validator.md`, `commands/validate.md`); write `docs/conversations/conversation-1.md`; add
`README.md`. Note: the orchestrating agent writes the conversation file (subagents can't see the
transcript); the validator's gate only verifies it exists.

**user** — approved.
