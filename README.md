# Document Processor Orchestration

Governance and SDD hub for the document/invoice processing product family: a **web
service** (Python/FastAPI), a **web app** (React + Vite SPA), and a **mobile app**
(Capacitor + React), coordinated by a single OpenAPI 3.1 contract owned here.

## Documents

- [`docs/feasibility.md`](docs/feasibility.md) — feasibility analysis: stack, topology,
  testing strategy, SDD environment, and the decision log (D1–D10).
- [`docs/conversations/`](docs/conversations/) — working-session transcripts
  (conversation-persistence gate, decision D10).

## Component repos (siblings)

| Repo | Role |
|------|------|
| `document-processor` | Web service (exists — continues evolving) |
| `document-processor-web` | Web app (new) |
| `document-processor-mobile` | Mobile app (new) |

## SDD workflow

This repo is the governance hub: constitution, product spec (Gherkin), OpenAPI contract,
glossary, ADRs, and conversations live here. Each component repo keeps its own
`.opencode/` agent/command set and CI, but contract changes here gate downstream PRs.

See [`docs/feasibility.md`](docs/feasibility.md) §8 for the SDD pipeline and §12 for the
decision log.
