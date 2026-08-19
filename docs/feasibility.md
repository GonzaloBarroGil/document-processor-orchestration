# Document Processor Orchestration — Feasibility Analysis

> **Date:** 2026-08-12
> **Status:** Draft v0.3 — decisions accepted, Hosting + cost governance resolved
> **Audience:** HITL (human-in-the-loop) + SDD agents
> **Scope:** Orchestrating a web service + web app + mobile app around the document/invoice processing domain
> **Source material:** `../document-processor` (mature Python/FastAPI service), `../ICES` (SDD specs)

---

## 1. Executive Summary

We are consolidating the work already produced in `document-processor` (a production-grade
Python/FastAPI OCR service for invoices, tickets, and payment documents) and the early SDD
specs in `ICES` (invoice capture with mobile client, jurisdiction pluggability, replaceable
extractor/queue/storage components) into a single coordinated product family:

| Component    | Role                                                        |
|--------------|-------------------------------------------------------------|
| **Web service** | Ingestion API + async OCR/validation worker + storage lifecycle |
| **Web app**  | Back-office UI: document review, validation queue, admin/keys, dashboards |
| **Mobile app** | Capture/scan with offline queue and progressive sync        |

**Recommended approach (summary):**

1. **Repo topology:** this repo is the governance/SDD hub; each component lives in its own repo.
2. **Backend:** keep and extend **Python 3.12 + FastAPI** (reuse `document-processor`), driven by the
   open-source OCR mandate. Do **not** migrate to TypeScript.
3. **Web app:** **React + TypeScript + Vite** SPA, generated client from the shared OpenAPI contract.
4. **Mobile app:** **Capacitor + React** (same codebase as web where possible) for camera + offline queue;
   Flutter is the fallback if native camera performance becomes a hard requirement.
5. **Single source of truth:** one **OpenAPI 3.1 contract** owned by this repo, consumed by all three.
6. **Testing:** strict SDD with coverage gates per layer (unit + integration + automation/E2E + contract).

The dominant constraint is the **open-source mandate** and the **OCR ecosystem**, which anchor the
backend to Python. Everything downstream (contracts, tests, SDD process) is organized around that.

---

## 2. Source Material Assessment

### 2.1 `document-processor` — mature, reusable

| Asset | Status | Reuse decision |
|-------|--------|----------------|
| Hexagonal architecture (Ports & Adapters) | Implemented | **Keep** — the canonical pattern for the family |
| Pipe & Filter pipeline (Ingest → Preprocess → OCR → Parse → Validate → Persist) | Implemented | **Keep** |
| 9 features / 31 Gherkin scenarios (`docs/spec.md` v1.0) | Approved | **Keep** — becomes product spec input |
| Constitution v1.1 with SDD pipeline + quality gates | Approved | **Keep** — promoted to family-wide constitution |
| 11 ADRs (`docs/adr/`) | Approved | **Keep** — archived as-is |
| `mypy strict` + `ruff` + coverage gates (domain ≥90%, adapters ≥70%) | Enforced | **Keep** |
| BDD step definitions for all 9 features | Complete | **Keep** |
| Worker E2E with PostgreSQL testcontainers | Complete | **Keep** — template for the family E2E |
| **Gap:** no web/mobile clients (explicitly out of scope) | — | **New** — the web/mobile apps close this gap |

### 2.2 `ICES` — early SDD, partially conflicting

| Asset | Status | Reuse decision |
|-------|--------|----------------|
| `constitution.md` (jurisdiction extension, replaceable components, API stability, data portability, observability, AR compliance) | Draft | **Merge** principles into family constitution where compatible |
| `api-spec.yaml` (OpenAPI 3.0, `/v1/invoices`) | Draft | **Replace** with unified OpenAPI 3.1 contract (document-processor's `/api/v1/documents` + invoice fields) |
| `database-schema.sql` (invoices, extracted_data, extraction_audit, failed_extractions) | Draft | **Merge** audit-trail + failed-extraction ideas into document-processor schema |
| `mobile-contract.md` (Capacitor, IndexedDB offline queue, camera 1920×1080 min) | Draft | **Adopt** as the mobile app spec baseline |
| `extractor-interface.ts` (IExtractor + ExtractionHints/Result) | Draft | **Map** onto document-processor's `OCRPort` (already the Python equivalent) |
| `implementation-plan.md` | Draft | Superseded by this feasibility + new plan |
| **Conflict:** plans AWS Textract + BullMQ | Draft | **Reject** — violates the open-source mandate; keep PaddleOCR/EasyOCR + PG `SKIP LOCKED` |

### 2.3 Key synthesis

`document-processor` and `ICES` describe the **same product** at different maturity levels and from
different entry points (backend-first vs mobile-first). The orchestration challenge is **not** technical
merging of two codebases — it is unifying their *contracts and process* so one backend serves two clients
(web + mobile) without divergence.

---

## 3. Product Scope

### 3.1 The unified domain

- Capture a document image (invoice, ticket, payment receipt) via mobile or web upload.
- Preprocess (HEIC transcode, PDF rasterize), OCR (PaddleOCR primary / EasyOCR fallback), parse,
  validate against regional rules (AR first, pluggable), persist (PostgreSQL + MinIO).
- Review, search, list, and export via a web back-office.
- Offline-first capture on mobile with retry/dead-letter.

### 3.2 Component responsibilities

| Component | In scope (v1) | Out of scope (deferred) |
|-----------|---------------|-------------------------|
| Web service | Ingestion, status, list, image download, health, rate-limit + daily doc quota + kill-switch, auth, worker, storage lifecycle | Multi-tenant billing, webhooks, event stream |
| Web app | Document list/search, detail + parsed data review, validation queue (manual fix flag), API key admin, dashboard | Report builder, user management/RBAC UI, PDF liquidacion export |
| Mobile app | Camera capture, quality validation, offline queue (FIFO + exponential backoff), status polling | WebSocket push, on-device OCR (WASM) |

### 3.3 Non-functional requirements (from both sources)

- Open-source dependencies only (OSI-approved).
- Argentine compliance: 10-year retention, immutable audit trail, CUIT/CAE/IVA extraction, soft-delete.
- Content-addressed image keys (SHA-256) for dedup + portability.
- Observability: log jurisdiction, provider, confidence, latency, cache hits per extraction.
- API stability: breaking changes = new `/v{N+1}`; internal contracts free within major version.

---

## 4. Repository Topology

**Decision: governance hub (this repo) + one repo per component.**

```
document-processor-orchestration/   ← THIS REPO (SDD & governance only)
├── docs/
│   ├── feasibility.md              ← this document
│   ├── constitution.md             ← family-wide constitution (promoted from document-processor v1.1)
│   ├── spec/                       ← product + cross-component specs (Gherkin)
│   ├── contracts/                  ← OpenAPI 3.1 + JSON Schema (single source of truth)
│   ├── adr/                        ← cross-cutting ADRs (archived document-processor ADRs + new)
│   ├── glossary.md                 ← ubiquitous language
│   └── conversations/              ← working transcripts (like news-app-feasibility / cesar-orchestration)
├── .opencode/
│   ├── agents/                     ← SDD subagents (constitution-drafter, spec-writer, task-planner, implementer, validator)
│   └── commands/                   ← /constitution /spec /plan /implement /validate
└── README.md

# sibling repos (separate git repos)
document-processor/          # web service (exists — continues evolving)
document-processor-web/      # web app (new)
document-processor-mobile/   # mobile app (new)
```

**Rationale:**
- Matches the existing `news-app-feasibility` + `news-app` and `cesar-orchestration` + `cesar-web` patterns.
- Contracts/SDD artifacts live in one place and are versioned independently of code.
- Each component repo keeps its own CI, but **contract changes here gate downstream PRs** (see §9).
- No git submodules: they couple histories and complicate the HITL commit governance (§8.4).

**Rejected alternatives:**
- **Monorepo:** simpler atomic changes, but heavier for a family with a Python backend + two TS clients,
  and inconsistent with the existing separated-repo pattern.
- **Submodules:** contract drift and history coupling outweigh single-checkout convenience.

---

## 5. Architecture

### 5.1 Contract-first, hexagonal everywhere

```
                    ┌────────────────────────────────────────────┐
                    │   ORCHESTRATION REPO — OpenAPI 3.1 + specs   │
                    │   (single source of truth, versioned)        │
                    └──────┬──────────────────┬────────────────────┘
              generates    │                  │               generates
        ┌───────────────▼───┐    ┌────────────▼────────┐   ┌─────────▼─────────┐
        │   Web service     │    │   Web app (SPA)     │   │   Mobile app      │
        │  Python/FastAPI   │    │  React + Vite + TS  │   │  Capacitor + React│
        │  + worker         │    │  generated client   │   │  generated client │
        └───────────────────┘    └─────────────────────┘   └───────────────────┘
```

- **OpenAPI 3.1** is the contract. The web service *is* the implementation of it; web + mobile *consume* it.
- TypeScript clients generated with `openapi-typescript` + `openapi-fetch` (or `orval`).
- Server schemas validated at runtime by FastAPI/Pydantic (already true).
- Cross-component DTOs never hand-written twice.

### 5.2 Web service internal architecture (unchanged)

Hexagonal + Pipe & Filter, as documented in `document-processor/docs/constitution.md` §3. The only
additions needed are endpoints the new clients require (e.g., explicit "manual fix / review" and
export endpoints surfaced for the web app) — introduced as a new `/v2/` or additive `/v1/` paths per
the API-stability rule.

### 5.3 Data model

Start from `document-processor`'s models, folding in `ICES`'s two valuable ideas:
1. **`extraction_audit`** immutable trail (user_id, timestamp, confidence, provider, action) — satisfies
   the observability + AR compliance requirements.
2. **`failed_extractions`** dead-letter with `retry_count` — aligns with the mobile offline-sync retry model.

---

## 6. Stack Evaluation & Recommendation

### 6.1 Backend

| Criterion (weight) | Python 3.12 + FastAPI | TypeScript + NestJS/Fastify |
|--------------------|----------------------|------------------------------|
| OCR ecosystem, open source (30%) | **5** — PaddleOCR/EasyOCR, Tesseract, Surya | 1 — mostly proprietary (Textract/Vision) or weak native bindings |
| Reuse of existing work (25%) | **5** — 11 ADRs, 9 features, full test suite, worker | 1 — ICES has specs only, no code |
| Type safety & tooling (15%) | 4 — `mypy strict` (3 documented OCR exemptions) | **5** — TS strict first-class |
| Async & concurrency (10%) | 4 — asyncio + FastAPI | **5** — event loop + BullMQ |
| Shared contracts with FE (10%) | 4 — OpenAPI generates TS clients | 5 — native TS DTO sharing |
| Language unification (10%) | 2 — three languages in family | **4** — one language for web+mobile+BE |
| **Weighted total** | **4.45** | **3.00** |

**Recommendation: keep Python/FastAPI.** The open-source OCR mandate (a constitutional non-negotiable
from `document-processor` §8) is decisive — there is no open-source, Spanish-capable, typed OCR library,
and the Python ecosystem owns OCR. `ICES`'s AWS Textract plan is rejected (proprietary, per-use cost,
lock-in); keep PaddleOCR/EasyOCR and treat any cloud OCR as a swappable adapter behind `OCRPort`
(which is exactly what `ICES`'s `IExtractor` intended).

### 6.2 Web app

| Criterion (weight) | React + Vite + TS (SPA) | Next.js (SSR/SSG) |
|--------------------|------------------------|-------------------|
| Testability & simplicity (35%) | **5** — Vitest + RTL + Playwright, no server | 4 — SSR testing surface is larger |
| Product fit (auth-gated back-office, 30%) | **5** — no public/SEO content | 2 — SSR/SEO value near zero |
| Reuse with mobile via Capacitor (20%) | **5** — SPA bundles directly into Capacitor | 3 — server components complicate webview reuse |
| Ecosystem & hiring (15%) | **5** | 5 |
| **Weighted total** | **4.85** | **3.30** |

**Recommendation: React + Vite + TypeScript SPA.** The back-office is fully authenticated with no SEO
surface, so SSR adds cost without benefit. A plain SPA is also the same bundle Capacitor consumes,
maximizing web↔mobile code reuse.

### 6.3 Mobile app

| Criterion (weight) | Capacitor + React | React Native | Flutter |
|--------------------|-------------------|--------------|---------|
| Code reuse with web app (30%) | **5** — same components, hooks, generated client | 3 — RN Web partial | 1 — Dart, no reuse |
| Camera quality & control (25%) | 3 — plugin quality presets (1920×1080 min met) | **5** — mature native camera | **5** — excellent camera |
| Offline-first (IndexedDB/SQLite) (20%) | **4** — IndexedDB (per ICES), 50-item queue fine | 4 — SQLite/WatermelonDB | **5** — sqflite/drift |
| Test automation maturity (15%) | 4 — Playwright (web) + Maestro/Detox | 4 — Detox | **5** — integration_test + patrol |
| Team/language consistency (10%) | **5** — TS everywhere | 5 | 2 — Dart |
| **Weighted total** | **4.35** | **4.15** | **3.35** |

**Recommendation: Capacitor + React**, directly reusing the web app, with the ICES mobile-contract as
the spec baseline (camera min 1920×1080, JPEG q85%, ≤5 MB, IndexedDB ≤50 pending, FIFO + exponential
backoff + dead-letter). **Trigger to reconsider:** if production camera capture shows unacceptable
quality/latency or a hard need for on-device OCR emerges, evaluate **Flutter** as the fallback (best
native camera + offline, at the cost of language divergence).

### 6.4 Shared contracts

| Option | Verdict |
|--------|---------|
| `openapi-typescript` + `openapi-fetch` | **Adopt** — minimal, type-safe generated client for web + mobile |
| `orval` (React Query integration) | Adopt only if we need per-hook codegen with TanStack Query |
| Hand-written DTOs | Rejected — drift risk, violates single-source-of-truth |

### 6.5 Infrastructure (per component)

| Concern | Choice | Notes |
|---------|--------|-------|
| Hosting | Self-host VPS (docker-compose), 4 vCPU / 8 GB | provider-agnostic; Hetzner CX42 suggested — see D11 |
| Edge / TLS | Cloudflare free + Caddy (auto-TLS) | hides origin, DDoS/edge rate-limit — see D12 |
| Database | PostgreSQL 16 (docker) | shared by web service only (apps never touch DB); migration path to managed |
| Object storage | MinIO (S3-compatible, self-hosted) | content-addressed SHA-256 keys; presigned URLs only; migration path to R2 |
| Queue | PostgreSQL `SKIP LOCKED` | upgrade path to Redis/AMQP documented (ADR-001) |
| Containers | Docker + compose | service only; web/mobile build separately |
| Package mgmt (BE) | uv | existing |
| Package mgmt (FE/mobile) | pnpm | TS workspaces if any shared package emerges |

---

## 7. Testing Strategy (strict SDD + high coverage)

### 7.1 Test pyramid, per component

```
        ┌─────────────────────────────────────────────────────┐
        │ E2E / automation (Playwright web, Maestro/Detox mobile, │
        │            worker E2E with testcontainers)              │
        ├─────────────────────────────────────────────────────────┤
        │ Integration (FastAPI TestClient+testcontainers, MSW,    │
        │            Capacitor plugin mocks, contract tests)       │
        ├─────────────────────────────────────────────────────────┤
        │ Unit (pytest domain/pipeline, Vitest components/hooks,   │
        │       Jest-free pure TS)                                 │
        └─────────────────────────────────────────────────────────┘
```

### 7.2 Per-component matrix

| Layer | Web service (Python) | Web app (React) | Mobile app (Capacitor) |
|-------|----------------------|-----------------|------------------------|
| Unit | pytest (`domain` ≥90%, `adapters` ≥70%) | Vitest + React Testing Library (≥80%) | Vitest for pure logic/queue (≥80%) |
| Integration | FastAPI `TestClient` + testcontainers (Postgres, MinIO); repository/worker integration | MSW mocked API against generated types; route + data-fetch tests | Capacitor plugin mocks; offline-queue state machine; IndexedDB integration |
| Automation/E2E | Worker E2E (queue → OCR → persist); API contract via `schemathesis` | Playwright E2E (login → list → review → export) | Maestro (or Detox) native flows; Playwright against Capacitor web bundle |
| Contract | `schemathesis` + OpenAPI validation in CI | Type-safe client compile (catches contract drift) | Same generated client |
| Static | `mypy strict` + `ruff` | `tsc --noEmit` + ESLint | `tsc --noEmit` + ESLint |

### 7.3 Coverage gates (CI-enforced)

| Component | Unit | Integration | E2E |
|-----------|------|-------------|-----|
| Web service | domain ≥90%, adapters ≥70% (existing gate) | required, all marked `integration` | worker E2E required |
| Web app | ≥80% | required for data-fetch/hooks | smoke on critical paths |
| Mobile app | ≥80% (logic) | required for offline queue | smoke (capture→offline→sync) |

**Automation posture:** every E2E suite must run in CI without manual device farms for web; mobile E2E
runs on emulators/simulators in CI (Maestro/Detox). No "run locally only" tests accepted.

### 7.4 Contract testing (the glue across repos)

- **Single OpenAPI file** in this repo is the source of truth; both TS clients and the Python server are
  validated against it in CI.
- **`schemathesis`** property-tests the live service against the OpenAPI spec (fuzz on top of unit tests).
- **Consumer-driven drift guard:** when the contract changes here, CI regenerates TS clients and compiles
  web + mobile to prove no breakage (see §9).

---

## 8. SDD Environment

### 8.1 Pipeline (promoted from `document-processor/docs/constitution.md` §6)

```
Constitution ──► Spec ──► Plan ──► Tasks ──► Implementation ──► Validation
     │              │        │         │            │               │
     └── HITL ✓ ────┴── HITL ✓ ──┴── HITL ✓ ──┴── HITL ✓ ──┴── HITL ✓ ──┘
```

Each phase has a dedicated subagent, and each phase is gated by HITL approval before the next begins.
Agents operate strictly within their phase's boundaries.

### 8.2 Artifact ownership across repos

| Artifact | Owner repo | Notes |
|----------|-----------|-------|
| Constitution (family-wide) | **this repo** | promoted from document-processor v1.1 + ICES merge |
| Product spec (Gherkin) | **this repo** | cross-component scenarios |
| OpenAPI contract + JSON Schema | **this repo** | versioned; breaking changes = new `/vN` |
| Glossary / ubiquitous language | **this repo** | single source of truth |
| Cross-cutting ADRs | **this repo** | archived document-processor ADRs + new |
| Component specs/plans/tasks | each component repo | must conform to contract + constitution |
| Conversations | **this repo** | working transcripts |

### 8.3 Agents (carried over from document-processor `.opencode/agents/`)

The family Constitution is hub-owned (§8.2); component repos enter the pipeline at **Spec** and
conform to it. Roster per repo:

| Agent | Phase | Where | Scope |
|-------|-------|-------|-------|
| `constitution-drafter` | Constitution | **hub only** | proposes amendments as Change Artifacts for HITL |
| `spec-writer` | Spec | all component repos | feature specs + acceptance scenarios |
| `task-planner` | Plan | all component repos | decomposes spec into atomic tasks, ADRs |
| `implementer` | Tasks/Implementation | all component repos | code + tests, one task at a time |
| `validator` | Validation | all component repos | runs lint/typecheck/test/coverage, reports; also verifies the conversation-persistence gate (D10) |

The web + mobile repos carry a TS-flavored subset (`spec-writer`, `task-planner`, `implementer`,
`validator` — `tsc`/ESLint/Vitest/pnpm) and no `constitution-drafter`.

Plus two new cross-component agents (proposed, not yet added):
- `contract-designer` — owns the OpenAPI contract and generated-client build.
- `e2e-engineer` — owns Playwright/Maestro automation suites.

**Note:** `document-processor`'s component-level constitution and `constitution-drafter` were
retired after the family constitution absorbed the backend-only quality points (v1.1); its
`.opencode/` agents/commands now reference the hub constitution directly (see
`docs/conversations/conversation-8.md`).

### 8.4 Git governance (unchanged non-negotiable)

AI agents may read repo state; **only HITL commits**. No AI-initiated `add/commit/push/merge/rebase/reset`.
This applies to **all four repos** — enforced by convention and CI branch protection where hosted.

### 8.5 Constitution change flow

Impassable constraint → documented Change Artifact (constraint, blocker, proposed amendment, impact
assessment) → HITL approve/reject → version bump (v1.2, v1.3…). No silent bypasses.

### 8.6 Conversation persistence (mandatory gate)

Given frequent unexpected shutdowns, **no HITL gate may pass unless the working
session has been persisted.** Concretely:

- After every Validation report, the **orchestrating agent** writes
  `docs/conversations/conversation-N.md` (raw transcript + decisions table). It is the
  only agent with full-session context — subagents run in fresh contexts and cannot see
  the transcript.
- The `validator` agent's output checklist ends with a final gate:
  **"Conversation persisted? (path)"** — a missing file blocks the gate.
- Recovery path is the `resume-session` skill (`opencode export` → resume).

This is decision **D10** (§12).

---

## 9. CI/CD

### 9.1 Per-component CI (each repo)

- **Web service:** `lint → typecheck (mypy strict) → unit → integration (testcontainers) → worker E2E → build`.
  Mirror of the existing `Makefile` `all` target.
- **Web app:** `tsc → eslint → vitest (coverage) → Playwright E2E → build`.
- **Mobile app:** `tsc → eslint → vitest (coverage) → Maestro/Detox on emulator → build`.

### 9.2 Orchestration-repo CI (this repo)

- Markdown lint (documents), `spectral` lint on OpenAPI, link checker, glossary consistency.
- **Contract-change workflow:** PR touching `contracts/` triggers downstream CI on the web-service,
  web, and mobile repos against the new spec (regenerate client, compile, run contract tests). A
  breaking contract change cannot merge until all consumers are green — enforcing API-stability by construction.

---

## 10. Risks & Mitigations

| Risk | Impact | Mitigation |
|------|--------|------------|
| OCR libraries untyped → mypy friction | Medium | Documented exemption already exists (ADR-011); keep the boundary-parse compensating control |
| Contract drift across 3 repos | High | Single OpenAPI + generated clients + cross-repo contract-change CI gate |
| Mobile E2E flakiness on emulators | Medium | Prefer Maestro (declarative); keep E2E smoke-scoped; retry-with-report |
| Web↔mobile code reuse over-promised | Medium | Define a shared UI package boundary early; accept partial reuse |
| Three-language cognitive load | Medium | Confine Python to the service; keep web+mobile TS-idiomatic |
| ICES Textract/BullMQ divergence | Low (already rejected) | Retain PaddleOCR/EasyOCR + PG queue; Textract as optional adapter only |
| Public abuse → cost runaway (public repos, paid infra) | High | D12: auth-gated OCR, 100 docs/day cap + per-key quotas, Cloudflare edge, storage lifecycle, kill-switch alerts |
| Scope creep (reports/RBAC/WS) | Medium | v1 scope locked in §3.2; defer list explicit |

---

## 11. Open Questions

### 11.1 Hosting — resolved (D11)

**Decision: self-host VPS (docker-compose) as the v1 default**, with managed services
documented as the scale/migration path. Rationale: minimize cost (flat VPS beats
usage-based managed at 100–1000 docs/day), no Argentine data-residency block, and
100–1000 docs/day is comfortably handled by a single 4 vCPU / 8 GB machine.

Reference topology (provider-agnostic; Hetzner CX42 suggested):

| Layer | Choice |
|-------|--------|
| Edge | Cloudflare free (hides origin, DDoS/edge rate-limit) |
| VPS | 4 vCPU / 8 GB (provider-agnostic; Hetzner CX42 suggested) |
| Reverse proxy | Caddy (auto-TLS) |
| API + worker | FastAPI + worker containers (docker-compose) |
| PostgreSQL | docker + WAL/nightly `pg_dump`; offsite via rclone |
| Object storage | MinIO (S3-compat); presigned URLs only; offsite backup to B2/R2 |
| Monitoring | compose healthchecks + Uptime Kuma |

**Scalability / migration ladder:**

1. **Vertical** → 16 GB VPS (first move).
2. **Horizontal** → split the OCR worker onto its own host; PG/MinIO stay.
3. **GPU** → dedicated worker node only if OCR latency becomes a constraint (optional).
4. **Managed escape hatch** → MinIO→Cloudflare R2, self-Postgres→Neon/Render,
   worker→Fly.io — drop-in because S3-compat + Postgres are standard, honoring the
   open-source/portability mandate.

### 11.2 Cost & abuse guardrails — resolved (D12)

Public repos (portfolio) + a paid, non-free-tier environment require explicit guardrails
so a public API cannot burn budget. OCR/upload is auth-gated; cost is bounded by quotas.

| Guardrail | Mechanism |
|-----------|-----------|
| Auth-gated ingestion | OCR/upload requires JWT or API key (D9) — no anonymous OCR |
| Edge protection | Cloudflare free in front of the VPS |
| Rate limiting | per-key/IP limits on `/ingest` and image download |
| Daily document cap | 100 docs/day global cap + per-key quota, enforced by the worker before dequeuing |
| Storage lifecycle | retention/archive; presigned URLs only (no public bucket) |
| Budget kill-switch | spend alerts on offsite backup (B2/R2); queue halts above the daily cap |
| Secrets hygiene | `.env` gitignored + `.env.example`; secrets via env/CI; no creds in docs or conversations |

### 11.3 Remaining open questions

1. **Manual review workflow** — is human-in-the-loop review (confirm/edit extracted
   fields) a v1 feature? ICES implied it; document-processor has no review endpoint yet.
2. **License** — open-source mandate says "TBD in Spec phase" — pick (e.g., AGPL-3.0 vs
   MIT) before public release.
3. **On-device OCR (WASM)** — explicitly deferred; revisit as a mobile quality trigger.

**Resolved:** Auth (D9), Hosting (D11), and cost & abuse guardrails (D12).

---

## 12. Decision Log

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
| D9 | Authentication: JWT (web + mobile) with rotating refresh tokens, roles `ADMIN`/`REVIEWER`, RBAC middleware; API keys retained for machine clients | Accepted |
| D10 | Conversation persistence: every Validation stage / HITL gate persists the session to `docs/conversations/` before passing | Accepted |
| D11 | Hosting: self-host VPS (docker-compose) as v1 default; provider-agnostic (4 vCPU / 8 GB, Hetzner suggested); managed services as the documented migration path | Accepted |
| D12 | Public exposure & cost governance: Cloudflare free edge, auth-gated ingestion, rate limits, 100 docs/day global cap + per-key quotas, storage lifecycle + signed URLs, budget kill-switch, secrets hygiene | Accepted |

> Remaining open questions (§11.3): Manual review, License, On-device OCR (WASM).

---

## 13. Recommended Next Steps

1. ~~Draft the **family constitution**~~ → done: `docs/constitution.md` v1.0 (approved).
2. ~~Draft the **unified OpenAPI 3.1 contract**~~ → done: `docs/contracts/openapi.yaml` (approved).
3. ~~Stand up the three component repos with the shared `.opencode/` agent/command set~~ → scaffolded: `document-processor-web` + `document-processor-mobile` (TS-flavored); `document-processor` retains its Python set.
4. ~~Write the **product spec** (Gherkin)~~ → done: `docs/spec/product.md` (approved).
5. ~~Establish the **contract-change CI gate** before any app code depends on the contract~~ → scaffolded: dormant workflows in hub + 3 component repos (activate on push).

---

**Status:** Draft v0.3 — decision log accepted (D1–D12); Hosting + cost governance resolved.
