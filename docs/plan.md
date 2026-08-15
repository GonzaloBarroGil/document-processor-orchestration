# Cross-Component Implementation Plan v1.0

## Document Processor — Product Family

> **Status:** Approved v1.0
> **Date:** 2026-08-14
> **Scope:** decompose the approved product spec (`docs/spec/product.md`, F1–F18) into ordered,
> testable implementation work across the three component repos + this governance hub.
> **Inputs:** `docs/constitution.md`, `docs/spec/product.md`, `docs/contracts/openapi.yaml`, `docs/feasibility.md`.

---

## 1. Scope & Strategy

The backend features **F1–F9 are already implemented** in `document-processor` (mature, 37-task plan
already delivered). This plan therefore covers:

| Surface | Repo | Nature of work |
|---------|------|----------------|
| Web service | `document-processor` | **Additive** — auth (D9), review/export endpoints, audit + dead-letter tables, daily quota (D12) |
| Web app | `document-processor-web` | **New** — React + Vite SPA (F10–F15) |
| Mobile app | `document-processor-mobile` | **New** — Capacitor + React (F16–F17) |
| Governance | `document-processor-orchestration` | **Infra** — component `.opencode/` scaffolding, contract-change CI gate, glossary |

**Ordering strategy (contract-first):** the OpenAPI contract is already approved and is the single
source of truth (D5). Work proceeds **web service first** (it is the contract implementation), then
**generated TS clients**, then **web app / mobile app** consume those clients. This lets both clients
develop against real, contract-typed endpoints and prevents the drift risk called out in
`docs/feasibility.md` §10.

---

## 2. Component Changes

### 2.1 Web service — additive changes (`document-processor`)

Promoted endpoints stay as-is (API-stability rule). New work maps to the contract paths added in the
Spec phase:

| Concern | Contract paths | Spec feature |
|---------|----------------|--------------|
| JWT auth + refresh + roles | `/auth/login`, `/auth/refresh`, `/auth/me`, `/auth/logout` | F10 |
| Manual review | `PATCH /documents/{id}/review`, `GET /review/queue` | F12 |
| Export | `GET /documents/{id}/export` (JSON + CSV) | F13 |
| Daily quota | enforced on `POST /documents` + worker dequeue | F18 |

Internal (non-API) additions: `users`/`refresh_tokens` tables (D9), `extraction_audit` immutable
trail + `failed_extractions` dead-letter (D8, ICES merge), `user_id` ownership + manual-fix fields on
`documents`.

### 2.2 Web app — new (`document-processor-web`)

React + TypeScript + Vite SPA. Auth-gated back-office with no SSR/SEO surface (feasibility §6.2).
Consumes the contract via a generated `openapi-fetch` client. Features F10–F15.

### 2.3 Mobile app — new (`document-processor-mobile`)

Capacitor + React, reusing web-app components where possible (feasibility §6.3). Features F16–F17
(camera capture + offline queue with FIFO / exponential backoff / dead-letter).

---

## 3. Data Model Deltas

Deltas apply to the `document-processor` PostgreSQL schema. Existing tables (`documents`,
`api_keys`, `storage_alerts`) are retained unchanged except where noted.

```sql
-- users (D9) — human users for web + mobile
CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    username        VARCHAR(100) NOT NULL UNIQUE,
    password_hash   VARCHAR(255) NOT NULL,             -- argon2id
    role            VARCHAR(20) NOT NULL DEFAULT 'REVIEWER',  -- ADMIN | REVIEWER
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- refresh_tokens (D9) — rotating refresh tokens
CREATE TABLE refresh_tokens (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    token_hash      CHAR(64) NOT NULL UNIQUE,          -- SHA-256, clear-text shown once
    expires_at      TIMESTAMPTZ NOT NULL,
    revoked         BOOLEAN NOT NULL DEFAULT FALSE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- extraction_audit (D8 / ICES merge) — immutable trail, observability + AR compliance
CREATE TABLE extraction_audit (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    document_id     UUID NOT NULL REFERENCES documents(id),
    user_id         UUID REFERENCES users(id),         -- NULL for machine clients
    provider        VARCHAR(50) NOT NULL,              -- paddle | easyocr
    confidence      FLOAT NOT NULL,
    action          VARCHAR(50) NOT NULL,              -- OCR | VALIDATE | REVIEW_APPROVE | ...
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- failed_extractions (D8) — dead-letter with retry accounting
CREATE TABLE failed_extractions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    document_id     UUID NOT NULL REFERENCES documents(id),
    retry_count     INT NOT NULL DEFAULT 0,
    last_error      TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- daily_usage (D12) — global + per-key daily ingestion counters
CREATE TABLE daily_usage (
    usage_date      DATE NOT NULL,
    scope           VARCHAR(20) NOT NULL,              -- global | key:<prefix>
    count           INT NOT NULL DEFAULT 0,
    PRIMARY KEY (usage_date, scope)
);

-- documents (delta) — ownership + manual review state
ALTER TABLE documents
    ADD COLUMN user_id        UUID REFERENCES users(id),
    ADD COLUMN reviewed       BOOLEAN NOT NULL DEFAULT FALSE,
    ADD COLUMN reviewed_by    UUID REFERENCES users(id),
    ADD COLUMN reviewed_at    TIMESTAMPTZ,
    ADD COLUMN edited_fields  JSONB;                   -- manual-fix overrides (F12)

CREATE INDEX idx_documents_user ON documents(user_id, created_at DESC);
```

---

## 4. Package Structure (new apps)

```
document-processor-web/
├── src/
│   ├── api/                 # generated openapi-fetch client (never hand-edited)
│   ├── auth/                # session store, login/refresh hooks, route guards, RBAC
│   ├── features/
│   │   ├── documents/       # list/filter/detail (F11)
│   │   ├── review/          # review queue + manual fix (F12)
│   │   ├── export/          # JSON/CSV export (F13)
│   │   ├── api-keys/        # ADMIN key administration (F14)
│   │   └── dashboard/       # dashboard (F15)
│   ├── components/          # shared UI (tables, pagination, status badges)
│   ├── lib/                 # pure helpers (parsing, formatters)
│   └── main.tsx
├── tests/
│   ├── unit/                # Vitest + RTL
│   ├── integration/         # MSW against generated types
│   └── e2e/                 # Playwright (login → list → review → export)
├── openapi-typescript.config.ts
├── vite.config.ts
├── package.json
└── README.md

document-processor-mobile/
├── src/
│   ├── api/                 # same generated client
│   ├── capture/             # camera plugin wrapper + quality validation (F16)
│   ├── queue/               # offline queue state machine (F17)
│   │   ├── store.ts         #   IndexedDB (≤50 pending)
│   │   ├── backoff.ts       #   exponential backoff
│   │   └── dead-letter.ts   #   terminal state + manual retry
│   ├── sync/                # progressive sync + status polling
│   └── components/          # reused from web app
├── tests/
│   ├── unit/                # Vitest (pure queue logic ≥80%)
│   ├── integration/         # IndexedDB + plugin mocks
│   └── e2e/                 # Maestro/Detox (capture → offline → sync)
├── capacitor.config.ts
├── package.json
└── README.md
```

---

## 5. Task Breakdown

### W — Web service (`document-processor`)

| ID | Task | Deps | Deliverable | Spec |
|----|------|------|-------------|------|
| W0.1 | Auth domain: `User` model + `AuthPort` | — | Pydantic models + ABC | F10 |
| W0.2 | JWT service (access + rotating refresh, argon2id) | W0.1 | token issue/verify/rotate | F10 |
| W0.3 | `users` + `refresh_tokens` migration | W0.1 | Alembic migration | F10 |
| W0.4 | `extraction_audit` + `failed_extractions` + `daily_usage` migrations | — | Alembic migration | F18 / D8 |
| W0.5 | `documents` delta migration (user_id, review fields) | W0.3 | Alembic migration | F12 |
| W1.1 | Auth API: `/auth/login`, `/auth/refresh`, `/auth/me`, `/auth/logout` | W0.2, W0.3 | 4 endpoints | F10 |
| W1.2 | RBAC dependency (`ADMIN`/`REVIEWER`) | W1.1 | FastAPI `Depends` guards | F10 |
| W1.3 | Review service (approve / request_changes / edited fields) | W0.5 | domain service | F12 |
| W1.4 | `PATCH /documents/{id}/review` + `GET /review/queue` | W1.2, W1.3 | 2 endpoints | F12 |
| W1.5 | Export service + `GET /documents/{id}/export` (JSON/CSV) | W0.5 | 1 endpoint | F13 |
| W1.6 | Daily quota + per-key enforcement (worker + ingest) | W0.4 | quota service + 429 | F18 |
| W1.7 | Audit + dead-letter wiring into pipeline | W0.4 | worker writes audit/retry | D8 |
| W1.8 | Auth + review + export BDD features | W1.1–W1.6 | `.feature` + steps | F10–F13, F18 |
| W1.9 | Auth/review/export integration tests (testcontainers) | W1.1–W1.7 | pytest integration | F10–F13 |
| W1.10 | Worker E2E (ingest → OCR → validate → audit → review) | W1.7 | full-pipeline E2E | F12 |

### A — Web app (`document-processor-web`)

| ID | Task | Deps | Deliverable | Spec |
|----|------|------|-------------|------|
| A0.1 | Vite + React + TS scaffold (pnpm, ESLint, Vitest, RTL) | — | runnable app skeleton | — |
| A0.2 | Generate TS client from `openapi.yaml` | — | `src/api/` (typed) | F10–F15 |
| A0.3 | Auth session store + refresh interceptor | A0.2 | token lifecycle | F10 |
| A1.1 | Login view + route guards + role-aware nav | A0.3 | auth flow UI | F10 |
| A1.2 | Document list + filters + pagination | A0.2 | list view | F11 |
| A1.3 | Document detail (parsed fields, confidence, validation) | A1.2 | detail view | F11 |
| A1.4 | Review queue + manual-fix editor | A0.2 | review UI | F12 |
| A1.5 | Export view (JSON + CSV download) | A0.2 | export UI | F13 |
| A1.6 | API-key administration (ADMIN) | A0.2 | key CRUD UI | F14 |
| A1.7 | Dashboard (counts + recent activity) | A0.2 | dashboard view | F15 |
| A2.1 | MSW integration tests for data-fetch/hooks | A1.1–A1.7 | typed-mock tests | F11–F15 |
| A2.2 | Playwright E2E (login → list → review → export) | A1.1–A1.7 | E2E suite | F11–F13 |

### M — Mobile app (`document-processor-mobile`)

| ID | Task | Deps | Deliverable | Spec |
|----|------|------|-------------|------|
| M0.1 | Capacitor + React scaffold (reuse web components) | — | runnable app skeleton | — |
| M0.2 | Generate TS client from `openapi.yaml` | — | `src/api/` (typed) | F16–F17 |
| M1.1 | Camera capture wrapper (1920×1080, JPEG ≤5 MB, quality check) | M0.1 | capture module | F16 |
| M1.2 | Offline queue store (IndexedDB, ≤50 pending) | M0.1 | `queue/store.ts` | F17 |
| M1.3 | FIFO upload + exponential backoff | M1.2 | `queue/backoff.ts` | F17 |
| M1.4 | Dead-letter state + manual retry surface | M1.2 | `queue/dead-letter.ts` | F17 |
| M1.5 | Sync + status polling (auth via JWT) | M0.2, M1.3 | `sync/` | F17 |
| M2.1 | Vitest unit tests (queue state machine ≥80%) | M1.2–M1.4 | logic tests | F17 |
| M2.2 | Plugin-mock + IndexedDB integration tests | M1.1, M1.5 | integration tests | F16–F17 |
| M2.3 | Maestro/Detox E2E (capture → offline → sync) | M1.1–M1.5 | emulator E2E | F16–F17 |

### X — Governance hub (`document-processor-orchestration`)

| ID | Task | Deps | Deliverable | Spec |
|----|------|------|-------------|------|
| X1.1 | Component `.opencode/` agent + command scaffolding (3 repos) | — | shared SDD toolset (§13 step 3) | — |
| X1.2 | Contract-change CI gate (regenerate clients → compile all consumers) | — | cross-repo CI workflow (§13 step 5) | — |
| X1.3 | `docs/glossary.md` (ubiquitous language) | — | glossary doc (§13) | — |

**Total: 33 tasks across 4 work streams (W: 10, A: 11, M: 8, X: 3).**

---

## 6. Dependency Order

```
X1.1 (scaffold) ─────────────────────────────────────────────► all streams

W0.1 ─► W0.2 ─► W0.3 ─► W0.5 ─► W1.1 ─► W1.2 ─► W1.4 ─► W1.8
                                    │          │
W0.4 ─► W1.6 ─► W1.7 ───────────────┘          │
                                    │          │
W1.1 ─► W1.5 (export)                └── W1.8 ─► W1.9 ─► W1.10

W1.1 (auth API live) ─► A0.2 (client) ─► A0.3 ─► A1.1 ─► A2.x
                        └─► M0.2 (client) ─► M1.5 ─► M2.x
```

- **W stream is the critical path** — both client streams wait on the auth + review + export
  endpoints and their generated clients (contract-first, D5).
- A and M streams proceed in parallel once the contract client is generated.
- X stream is independent of the contract and can run immediately.

---

## 7. Cross-Cutting Infrastructure (governance repo)

Maps to the two remaining §13 steps:

1. **Component `.opencode/` scaffolding (X1.1)** — copy the shared agent/command set
   (`constitution-drafter`, `spec-writer`, `task-planner`, `implementer`, `validator`, plus proposed
   `contract-designer` / `e2e-engineer`) into each component repo so downstream SDD phases run there.
2. **Contract-change CI gate (X1.2)** — a PR touching `docs/contracts/openapi.yaml` here triggers
   client regeneration + compile in web, mobile, and (via `schemathesis`) the web service. A breaking
   change cannot merge until all consumers are green (feasibility §9.2).
3. **Glossary (X1.3)** — `docs/glossary.md` is a constitutional deliverable (§13) currently missing;
   promote terms from `document-processor/docs/glossary.md` + the new auth/review/export terms.

---

## 8. ADR Index

New cross-cutting decisions recorded in `docs/adr/`:

| ADR | Title | Source |
|-----|-------|--------|
| 001 | Repository topology: governance hub + component repos | D1 |
| 002 | Contract-first with generated TypeScript clients | D5 |
| 003 | JWT authentication with rotating refresh tokens + RBAC | D9 |
| 004 | Human-in-the-loop review workflow | §11.3 Q1 |
| 005 | Public-exposure cost & abuse governance | D12 |

Archived `document-processor` ADRs (001–011) remain in that repo and are referenced, not copied.

---

**Status:** Approved v1.0.
