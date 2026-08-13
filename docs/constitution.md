# Constitution v1.0 — Document Processor Orchestration

## Family-Wide Constitution

> **Status:** Approved v1.0
> **Date:** 2026-08-13
> **Scope:** the document/invoice processing product family — web service, web app, mobile app
> **Provenance:** promotes `document-processor/docs/constitution.md` v1.1 to family-wide scope
> and merges the compatible principles from `ICES` (see `docs/feasibility.md` §2.2 for the merge map).

---

## 1. Identity & Scope

**Purpose:** a coordinated product family that ingests images of invoices, tickets, and payment
documents; extracts and validates structured information via OCR; persists data and stores images
with traceable identifiers; and exposes review/capture surfaces for humans on web and mobile.

**Components (in scope):**

| Component | Role |
|-----------|------|
| **Web service** | Ingestion API + async OCR/validation worker + storage lifecycle |
| **Web app** | Back-office UI: document review, validation queue, API-key admin, dashboards |
| **Mobile app** | Capture/scan with offline queue and progressive sync |

**Out of scope (deferred):** multi-tenant billing, webhooks, event streams, report builder,
user-management UI, on-device OCR (WASM) — see `docs/feasibility.md` §3.2.

---

## 2. Repository Topology (Family-Wide)

**One governance hub + one repo per component.** No monorepo, no git submodules.

```
document-processor-orchestration/   ← governance & SDD hub (this repo)
document-processor/                 ← web service
document-processor-web/             ← web app
document-processor-mobile/          ← mobile app
```

**Rules:**

- This repo owns the cross-cutting artifacts: this constitution, the product spec (Gherkin),
  the OpenAPI contract, the glossary, cross-cutting ADRs, and working transcripts
  (`docs/conversations/`).
- Component repos own their own code, specs, plans, tasks, and CI; they conform to the contract
  and this constitution.
- The **OpenAPI contract is the single source of truth**; it is versioned here and consumed by all
  three components (decision D5).
- Contract changes here gate downstream PRs via the contract-change CI workflow (see §11).

---

## 3. Technology Stack (Locked)

| Concern | Choice | Scope |
|---------|--------|-------|
| Language | Python 3.12+ | web service |
| Web framework | FastAPI | web service |
| OCR engine | PaddleOCR (primary), EasyOCR (fallback) | web service worker |
| Database | PostgreSQL 16+ | web service |
| Object storage | MinIO (S3-compatible) | web service |
| Job queue | PostgreSQL `SKIP LOCKED` | web service |
| Web app | React + TypeScript + Vite (SPA) | web app |
| Mobile app | Capacitor + React | mobile app |
| API contract | OpenAPI 3.1 (single, owned by hub) | all components |
| Client codegen | `openapi-typescript` + `openapi-fetch` | web + mobile |
| Containerization | Docker + docker-compose | web service |
| Package mgmt | uv (backend), pnpm (web/mobile) | per component |
| Linting | ruff (backend), ESLint (web/mobile) | per component |
| Type checking | mypy strict (backend), `tsc --noEmit` (web/mobile) | per component |
| BDD | behave or pytest-bdd | web service |
| TDD | pytest + pytest-asyncio; Vitest + RTL | per component |

**Hosting (decision D11):** self-host VPS (docker-compose) as the v1 default, provider-agnostic
(4 vCPU / 8 GB; Hetzner CX42 suggested), with managed services as the documented migration path.
See `docs/feasibility.md` §11.1. The portability rule below is the constitutional guarantee that
makes this posture safe to change.

---

## 4. Architecture Principles

### 4.1 Contract-first

- The **OpenAPI 3.1 contract is the single source of truth**. The web service *is* its
  implementation; the web app and mobile app *consume* it.
- TypeScript clients are **generated** from the contract. Cross-component DTOs are never
  hand-written twice.
- Server schemas are validated at runtime by FastAPI/Pydantic.

### 4.2 Hexagonal (Ports & Adapters)

- Domain core has **zero imports from infrastructure**. Adapters depend on ports, never the reverse.
- Ports include: `OCRPort`, `DocumentRepositoryPort`, `StoragePort`, `RegionValidatorPort`,
  `EventPort`, `ApiKeyRepositoryPort` (and the auth/user port added by D9).

### 4.3 Pipe & Filter pipeline

- The processing pipeline is `Ingest → Preprocess → OCR → Parse → Validate → Persist`.
- Each filter receives typed Pydantic input and returns typed Pydantic output; filters are
  composable and independently testable.

### 4.4 Worker isolation

- The queue worker is a **separate process** (not an in-process `asyncio.Task`).
- Worker and API communicate **only through the database**. This isolates CPU-bound OCR from API
  responsiveness and allows horizontal scaling (and a later GPU worker node).

---

## 5. Cross-Component Principles (merged from ICES)

| Principle | Concrete Rule |
|-----------|---------------|
| **API stability** | Breaking changes require a new `/v{N+1}`. Internal contracts are free within a major version. |
| **Data portability** | Images are content-addressed (SHA-256 keys) in S3-compatible storage. No vendor lock-in for storage or DB. |
| **Replaceable components** | Extractor, queue, and storage are swappable behind ports (`OCRPort`, queue, `StoragePort`). |
| **Observability** | Every extraction logs: jurisdiction, provider, confidence, latency, and cache hits. |
| **Compliance & audit** | Immutable `extraction_audit` trail (user, timestamp, confidence, provider, action); `failed_extractions` dead-letter with `retry_count`; soft-delete; 10-year retention. |

---

## 6. Design Principles (Non-Negotiable)

| Principle | Concrete Rule |
|-----------|---------------|
| **SOLID** | SRP per class; Open/Closed for region validators; Liskov for port interfaces; ISP for adapter contracts; DIP via ports |
| **Functional Core, Imperative Shell** | Validation, OCR text parsing, and data transformation are pure functions. I/O only in the adapter layer |
| **Repository Pattern** | All data access through repository interfaces. Never inline SQL in domain/service layer |
| **Dependency Injection** | FastAPI `Depends()` for wiring. No global singletons, no service locators |
| **Pydantic as Boundary** | Data crossing any module boundary is a Pydantic model, never a raw dict |
| **12-Factor App** | Config from env vars (pydantic-settings). Stateless processes. Explicit dependencies |
| **Composition over Inheritance** | Region validators composed from shared rule fragments |
| **Fail Fast** | Validate at the API boundary. Crash on unrecoverable worker errors. Never swallow exceptions silently |
| **Clean Code** | Descriptive names, small functions, no "what" comments — only "why" |

---

## 7. Quality Standards

### 7.1 Per-component matrix

| Layer | Web service (Python) | Web app (React) | Mobile app (Capacitor) |
|-------|----------------------|-----------------|------------------------|
| Unit | pytest — domain ≥90%, adapters ≥70% | Vitest + RTL ≥80% | Vitest (logic/queue) ≥80% |
| Integration | FastAPI `TestClient` + testcontainers | MSW against generated types | Capacitor plugin mocks; offline-queue state machine |
| E2E | Worker E2E (queue → OCR → persist); `schemathesis` | Playwright (login → list → review → export) | Maestro/Detox on emulators |
| Contract | `schemathesis` + OpenAPI validation in CI | Type-safe client compile | Same generated client |
| Static | ruff + mypy strict | ESLint + `tsc --noEmit` | ESLint + `tsc --noEmit` |

### 7.2 Gates

- **BDD:** every API endpoint has at least one `.feature` file; step definitions live in the
  component's `tests/bdd/`.
- **TDD:** unit tests written before implementation for domain logic (Red → Green → Refactor).
- **CI gate:** lint → typecheck → test → build must pass before any merge (per component).
- **Coverage:** domain ≥90%, adapters ≥70% (backend); ≥80% (web/mobile) — enforced in CI.
- **No "run locally only" tests:** every E2E suite must run in CI.

---

## 8. Development Process — SDD Pipeline

```
Constitution ──► Spec ──► Plan ──► Tasks ──► Implementation ──► Validation
     │              │        │         │            │               │
     └── HITL ✓ ────┴── HITL ✓ ──┴── HITL ✓ ──┴── HITL ✓ ──┴── HITL ✓ ──┘
```

| Phase | Output | HITL Role |
|-------|--------|-----------|
| **Constitution** | This document | Review, approve, amend |
| **Spec** | Feature specs, Gherkin scenarios, API contracts | Review, refine, approve |
| **Plan** | Task breakdown, file structure, interface definitions, ADRs | Review, reorder, approve |
| **Tasks** | Individual implementation units | Review each task's output |
| **Implementation** | Working code + tests + docs | Review diffs, run tests |
| **Validation** | Test reports, coverage, lint results | Review results, sign off |

**Rules:**

- Each phase may delegate to specialized subagents (see `docs/feasibility.md` §8.3 for the roster).
  Agents operate within the boundaries of the current phase. HITL gates block progression.
- **Conversation persistence (D10):** no HITL gate may pass unless the working session has been
  persisted to `docs/conversations/conversation-N.md` (raw transcript + decisions table). The
  orchestrating agent writes it; the `validator` gate verifies it exists before passing.
- **Constitution change:** if a principle proves to be a blocker, do not bypass it — propose a
  Constitution Change Artifact for HITL approval (§12).
- **Versioning:** each completed phase is committed to Git by HITL.

---

## 9. Git Governance

| Operation | AI Agent | HITL |
|-----------|----------|------|
| `git status` `git log` `git diff` `git show` | ✅ Allowed | — |
| `git add` `git commit` `git push` `git merge` `git rebase` `git reset` `git revert` `git cherry-pick` | ❌ Forbidden | ✅ Only HITL |

**Rule:** AI agents may read repository state at any time. Only HITL commits — across **all four
repos** (governance hub + three components). No AI-initiated destructive operations.

---

## 10. Open Source Mandate

- All dependencies must be open source (OSI-approved license).
- No proprietary OCR APIs (AWS Textract, etc.), no vendor lock-in for storage or DB.
- The projects themselves will be open source (license TBD in Spec phase — see `docs/feasibility.md`
  §11.3).

---

## 11. Public Exposure & Cost Governance (D12)

The repos are public (portfolio) and the environment is paid from day one, so the API must not be
drivable into unbounded cost. Enforced guardrails:

| Guardrail | Rule |
|-----------|------|
| Auth-gated ingestion | OCR/upload requires JWT or API key (D9) — no anonymous OCR |
| Edge protection | Cloudflare free in front of the VPS (hides origin, DDoS/edge rate-limit) |
| Rate limiting | per-key/IP limits on `/ingest` and image download |
| Daily document cap | 100 docs/day global cap + per-key quota, enforced before dequeuing |
| Storage lifecycle | retention/archive; presigned URLs only (no public bucket) |
| Budget kill-switch | spend alerts on offsite backup; queue halts above the daily cap |
| Secrets hygiene | `.env` gitignored + `.env.example`; secrets via env/CI; no creds in docs or conversations |

---

## 12. Constitution Amendment Process

1. An impassable blocker caused by a constitutional constraint must be documented as a
   **Change Artifact**.
2. The artifact describes: the constraint, the blocker, the proposed amendment, and impact
   assessment.
3. HITL reviews and approves or rejects.
4. If approved, the Constitution is versioned (v1.0 → v1.1 → …) and the change is recorded.

---

## 13. Documentation as First-Class Deliverable

| Phase | Documentation Artifact |
|-------|------------------------|
| Constitution | This document (`docs/constitution.md`) |
| Spec | Gherkin `.feature` files, OpenAPI contract, domain glossary |
| Plan | ADRs (`docs/adr/`), component/data-model diagrams |
| Tasks → Implementation | Docstrings (Google style), per-package README, migration notes |
| Validation | Doc review gate — missing/broken docs block sign-off |

**Rules:**

- API docs auto-generated from FastAPI (OpenAPI). Never hand-maintained.
- ADRs use the template: title, status, context, decision, consequences — stored in `docs/adr/`.
- Diagrams use Mermaid (text-based, version-controllable).
- Every public function/class gets a docstring; every feature gets a Gherkin file.
- `docs/glossary.md` defines domain terms — single source of truth for the ubiquitous language.

---

**Status:** Approved v1.0.
