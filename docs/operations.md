# Operations Guide — run locally, deploy, and manage DB & storage

> **Status:** Adopted
> **Scope:** the four repos — `document-processor` (web service), `document-processor-web` (SPA),
> `document-processor-mobile` (Capacitor app), governed by `document-processor-orchestration`.
> **Complements:** `docs/feasibility.md` §6.5/§11 (infrastructure decisions) and `docs/ci-cd.md`
> (branching + CI matrix).

This is the operational runbook. It assumes the deployment model decided in `feasibility.md` D11:
**self-host a single VPS with docker-compose** (PostgreSQL + MinIO + API + worker), Cloudflare free
in front, Caddy as the TLS/edge reverse proxy. Managed services are a documented migration path,
not the default.

---

## 1. Repos & process topology

| Component | Repo | Runtime | Ports (dev) |
|-----------|------|---------|-------------|
| API | `document-processor` | Python 3.12 + FastAPI (uvicorn) | `8000` |
| Worker (OCR) | `document-processor` | `docproc-worker` (separate process) | — |
| Web app | `document-processor-web` | React + Vite (static `dist/`) | `5173` |
| Mobile app | `document-processor-mobile` | Capacitor + React | `5173` (web) / native |

```
                         ┌──────────────┐
  client ──► Cloudflare ──► Caddy (TLS) ──► API :8000 (uvicorn server:app)
   (edge)                  (reverse proxy)   │
                                             ├─► Worker (docproc-worker, polls PG queue)
                                             ├─► PostgreSQL :5432 (SKIP LOCKED queue)
                                             └─► MinIO :9000 (objects), :9001 (console)
```

The API **ingests** documents and enqueues them; the **worker** runs the
`Preprocess → OCR → Parse → Validate → Persist` pipeline. The apps only talk to the API (never the
DB directly); images are served via presigned URLs.

---

## 2. Prerequisites

- **Backend:** Python 3.12+, Docker + docker-compose (for Postgres/MinIO and the full stack), `uv`
  or `pip`. The OCR engines (PaddleOCR/EasyOCR) pull model weights on first use.
- **Web/mobile:** Node.js 22+ and `pnpm` (pinned via `packageManager` — `pnpm@11.22.0`).
- **Mobile native build (optional):** JDK 21 (Capacitor 7) + Android SDK/Gradle.

---

## 3. Run locally

### 3.1 Backend — full Docker stack (simplest)

```bash
cd document-processor
cp .env.example .env        # review; set JWT_SECRET at minimum
docker compose up -d        # builds app+worker, starts postgres + minio
# apply the schema (inside the compose network, where host `postgres` resolves)
docker compose exec app alembic -c src/document_processor/adapters/persistence/postgresql/alembic.ini upgrade head
```

- API: `http://localhost:8000` · docs: `http://localhost:8000/docs` · health: `GET /api/v1/health`
- MinIO console: `http://localhost:9001` (`minioadmin` / `minioadmin` by default)

### 3.2 Backend — native (run only Postgres + MinIO in Docker)

Useful when you want hot reload or to avoid rebuilding the heavy OCR image:

```bash
# 1) infra only
docker compose up -d postgres minio

# 2) Python env
cd document-processor
python -m venv .venv && source .venv/bin/activate
pip install -e ".[dev]"

# 3) point config at localhost (not the `postgres`/`minio` service names)
export DATABASE_URL=postgresql+asyncpg://docproc:docproc@localhost:5432/docproc
export MINIO_ENDPOINT=localhost:9000
export JWT_SECRET=$(python -c "import secrets; print(secrets.token_urlsafe(64))")

# 4) migrate + run
make migrate                 # alembic upgrade head
uvicorn document_processor.adapters.web.server:app --reload --port 8000   # terminal 1
docproc-worker                                                             # terminal 2
```

> **OCR note:** PaddleOCR/EasyOCR download language models on first run and are CPU-heavy.
> Switch to the lighter engine with `OCR_PRIMARY_ENGINE=easyocr`. For pure API/contract work you can
> skip the worker entirely — the API stays up and documents just sit in `PENDING`.

### 3.3 Web app

```bash
cd document-processor-web
pnpm install
pnpm dev                    # http://localhost:5173
```

`src/api/client.ts` reads `VITE_API_URL` (default `http://localhost:8000`). The API's CORS allow-list
(defaults: `http://localhost:5173`, `:4173`, `:3000`) permits this cross-origin dev setup. For a
production build: `pnpm build` → serve the `dist/` directory (see §4.3).

### 3.4 Mobile app

```bash
cd document-processor-mobile
pnpm install
pnpm dev                    # web-bundle preview (browser), same VITE_API_URL default
```

Native (optional, JDK 21 required):

```bash
pnpm build
pnpm exec cap add android
pnpm exec cap sync android
cd android && ./gradlew assembleDebug
```

### 3.5 Seed accounts & API keys

The apps log in with JWT (`ADMIN`/`REVIEWER`); machine clients use API keys:

```bash
# create a login user (web/mobile)
docker compose exec app docproc-user create reviewer REVIEWER s3cret

# create / list / revoke API keys (machine clients)
docker compose exec app docproc-keys create my-app      # save the printed key — shown once
docker compose exec app docproc-keys list
docker compose exec app docproc-keys revoke <prefix>
```

(Native runs: invoke `docproc-user` / `docproc-keys` directly with `DATABASE_URL` set.)

### 3.6 Environment variables

All settings live in `document-processor/src/document_processor/core/config.py` (pydantic-settings,
loaded from `.env`). The important ones:

| Variable | Default | Purpose |
|----------|---------|---------|
| `DATABASE_URL` | `postgresql+asyncpg://docproc:docproc@postgres:5432/docproc` | PostgreSQL DSN |
| `MINIO_ENDPOINT` / `_ACCESS_KEY` / `_SECRET_KEY` | `minio:9000` / `minioadmin` / `minioadmin` | object storage |
| `MINIO_BUCKET` | `documents` | bucket (auto-created) |
| `OCR_PRIMARY_ENGINE` / `OCR_FALLBACK_ENGINE` | `paddle` / `easyocr` | OCR engines |
| `OCR_CONFIDENCE_THRESHOLD` | `0.7` | minimum confidence |
| `JWT_SECRET` | dev-only placeholder | **must change in prod** (≥32 chars) |
| `ACCESS_TOKEN_TTL_SECONDS` / `REFRESH_TOKEN_TTL_SECONDS` | 900 / 604800 | token lifetimes |
| `RATE_LIMIT_PER_MINUTE` | 60 | per-key ingest limit |
| `DAILY_DOCUMENT_CAP` | 100 | global daily ingestion cap |
| `MAX_IMAGE_SIZE_BYTES` | 10 MiB | max upload size |
| `STORAGE_HIGH_WATERMARK_PCT` / `STORAGE_CRITICAL_PCT` | 85 / 95 | expiry triggers |
| `STORAGE_EXPIRE_COMPLETED_DAYS` / `STORAGE_EXPIRE_CRITICAL_DAYS` | 90 / 30 | retention windows |
| `CORS_ALLOW_ORIGINS` | `["http://localhost:5173", ...]` | JSON array of allowed origins |

---

## 4. Production deployment

### 4.1 Topology (from `feasibility.md` D11)

Single VPS (4 vCPU / 8 GB suggested; provider-agnostic — Hetzner CX42 was the reference), with:

| Layer | Choice |
|-------|--------|
| Edge / DNS | Cloudflare free (hides origin, DDoS + edge rate-limit) |
| Reverse proxy | Caddy (auto-TLS via Let's Encrypt) |
| API + worker | two containers (docker-compose) |
| PostgreSQL | container + nightly `pg_dump`, offsite via rclone |
| Object storage | MinIO (S3-compatible), presigned URLs only |

### 4.2 Provision the VPS

1. Create the VM; open only `80`/`443` (Caddy) plus SSH. Block `5432`, `8000`, `9000`, `9001` at the
   firewall — they're reached only from the compose network / Caddy.
2. Install Docker + the compose plugin.
3. Point the DNS `A`/`AAAA` records at the VPS via Cloudflare (orange-cloud = proxied).

### 4.3 Reverse proxy — Caddy

Serve the SPA as static files and proxy the API. `Caddyfile` (template to create on the host):

```caddyfile
app.example.com {
    encode gzip
    root * /srv/web
    try_files {path} /index.html
    file_server
}

api.example.com {
    reverse_proxy 127.0.0.1:8000
}
```

Build the web app (`pnpm build`) and copy `dist/` to `/srv/web` (or add a build stage to the compose
file). The API's CORS list can then stay minimal (or empty) since SPA and API share the domain
family; keep it if the SPA calls `api.example.com` cross-origin.

### 4.4 Services — production compose

The repo's `docker-compose.yml` is **dev-oriented** (source mounts, default creds, exposed DB/console
ports). For production, adapt it (template):

```yaml
services:
  app:
    build: .
    restart: unless-stopped
    command: uvicorn document_processor.adapters.web.server:app --host 0.0.0.0 --port 8000
    env_file: .env.production
    depends_on:
      postgres: { condition: service_healthy }
      minio:   { condition: service_healthy }
    # no source mount, no published ports (Caddy reaches it on the compose network)

  worker:
    build: .
    restart: unless-stopped
    command: docproc-worker
    env_file: .env.production
    depends_on:
      postgres: { condition: service_healthy }
      minio:   { condition: service_healthy }

  postgres:
    image: postgres:16-alpine
    restart: unless-stopped
    environment: { POSTGRES_DB: docproc, POSTGRES_USER: docproc, POSTGRES_PASSWORD: <secret> }
    volumes: [pgdata:/var/lib/postgresql/data]
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U docproc"]
      interval: 3s

  minio:
    image: minio/minio:latest
    restart: unless-stopped
    command: server /data --console-address :9001
    environment: { MINIO_ROOT_USER: <secret>, MINIO_ROOT_PASSWORD: <secret> }
    volumes: [miniodata:/data]
    healthcheck:
      test: ["CMD", "mc", "ready", "local"]
      interval: 3s

volumes:
  pgdata:
  miniodata:
```

Keep the DB/console ports **unpublished** in prod (no `ports:` section) — they're only needed
on the compose network.

### 4.5 Secrets & config

- `.env` (or `.env.production`) is **gitignored**; `.env.example` is the committed template.
- Generate a real JWT secret: `python -c "import secrets; print(secrets.token_urlsafe(64))"`.
- Use distinct, strong credentials for PostgreSQL and MinIO.
- Never commit secrets; pass them via the env file (outside the repo) or the host's secret store.

### 4.6 Migrate-on-deploy

The schema is **not** auto-applied — run Alembic as a deploy step:

```bash
docker compose run --rm app alembic -c src/document_processor/adapters/persistence/postgresql/alembic.ini upgrade head
```

Then seed the initial admin and any API keys (§3.5). Deploys become: pull → build → migrate →
(re)create `app`/`worker`.

### 4.7 Cost & abuse guardrails (from `feasibility.md` D12)

| Guardrail | Mechanism |
|-----------|-----------|
| Auth-gated ingestion | uploads/OCR require JWT or API key (no anonymous OCR) |
| Edge protection | Cloudflare free in front of the VPS |
| Rate limit | `RATE_LIMIT_PER_MINUTE` per key/IP |
| Daily cap | `DAILY_DOCUMENT_CAP` (API enforces; worker re-checks before dequeuing) |
| Storage lifecycle | `docproc-lifecycle` sweep + retention days (§6.2) |
| Budget kill-switch | alert on storage usage / offsite backup spend; halt worker when cap exceeded |

Run the lifecycle sweep on a schedule (system cron or a container):

```bash
0 3 * * * cd /srv/document-processor && docker compose run --rm app docproc-lifecycle
```

### 4.8 Scaling ladder

1. **Vertical** → bump the VPS to 16 GB.
2. **Horizontal** → move the `worker` to its own host (PG + MinIO stay).
3. **GPU** → dedicated worker node if OCR latency becomes the constraint.
4. **Managed escape hatch** → MinIO→R2, self-Postgres→Neon/Render, worker→Fly.io (drop-in: S3-compat
   + Postgres are standard, preserving the open-source/portability mandate).

---

## 5. Database management

### 5.1 Migrations (Alembic)

Migrations live in
`document-processor/src/document_processor/adapters/persistence/postgresql/migrations/versions/`
(4 revisions as of writing). The config file is `.../postgresql/alembic.ini`, whose `env.py` pulls
the DSN from `settings.database_url` (env/`.env`).

```bash
cd document-processor
# current revision
alembic -c src/document_processor/adapters/persistence/postgresql/alembic.ini current

# apply pending
alembic -c src/document_processor/adapters/persistence/postgresql/alembic.ini upgrade head

# roll back one
alembic -c src/document_processor/adapters/persistence/postgresql/alembic.ini downgrade -1

# generate a new revision from model changes
alembic -c src/document_processor/adapters/persistence/postgresql/alembic.ini revision --autogenerate -m "description"
```

> `make migrate` is a shorthand for `upgrade head`.

### 5.2 Backups

Nightly logical dump + offsite copy (rclone) is the baseline (see `feasibility.md` D11):

```bash
# daily (cron)
docker compose exec -T postgres pg_dump -U docproc docproc | gzip > /backups/docproc-$(date +%F).sql.gz
rclone copy /backups remote:bucket/backups
```

Enable WAL archiving or a managed PG if you need point-in-time recovery.

### 5.3 Restore

```bash
docker compose exec -T postgres psql -U docproc docproc < /backups/docproc-2026-08-28.sql.gz   # after gunzip
```

### 5.4 Useful queries

```sql
-- counts by status
SELECT status, COUNT(*) FROM documents GROUP BY status;
-- recent activity
SELECT id, status, created_at FROM documents ORDER BY created_at DESC LIMIT 20;
-- failed extractions (dead-letter / retry)
SELECT * FROM failed_extractions ORDER BY created_at DESC LIMIT 20;
```

---

## 6. Storage management (MinIO)

### 6.1 Bucket & objects

The bucket (`MINIO_BUCKET`, default `documents`) is **auto-created** by `MinioStorage` on startup.
Images are stored under content-addressed keys; clients download via presigned URLs (the bucket is
not publicly accessible). Manage objects via the console (`:9001`) or `mc`.

### 6.2 Retention lifecycle

`docproc-lifecycle` (the `StorageLifecycleService`) expires images when storage exceeds the
watermarks:

| Setting | Effect |
|---------|--------|
| `STORAGE_HIGH_WATERMARK_PCT` (85) | expire `COMPLETED` images older than `STORAGE_EXPIRE_COMPLETED_DAYS` |
| `STORAGE_CRITICAL_PCT` (95) | also expire `VALIDATION_FAILED`, using `STORAGE_EXPIRE_CRITICAL_DAYS` |

Expired images are deleted from MinIO and marked `IMAGE_EXPIRED` in the DB (subsequent
`GET /documents/{id}/image` returns `410`).

> **Known limitation:** `MinioStorage.usage_pct()` computes usage against a hard-coded 10 GiB quota,
> not a configurable value. Tune the watermark thresholds to taste until it's parameterized.

### 6.3 Watermark alerts

Watch `usage_pct` (or the MinIO metrics) and alert when it crosses `STORAGE_HIGH_WATERMARK_PCT`
(`85`) and `STORAGE_CRITICAL_PCT` (`95`). In the reference topology this is a `Uptime Kuma` /
compose-healthcheck concern, not wired in the app itself.

### 6.4 Offsite backup

Mirror MinIO to a second S3 target with rclone (e.g., Cloudflare R2 / Backblaze B2):

```bash
rclone sync minio:documents r2:document-processor/documents
```

---

## 7. Health & troubleshooting

| Symptom | Likely cause |
|---------|--------------|
| API won't start | DB/MinIO unreachable, or schema not migrated (`alembic upgrade head`) |
| `401` from the apps | missing/expired JWT — create a user (`docproc-user`) and log in |
| `403` from machine clients | API key missing/revoked — `docproc-keys list` / `create` |
| `429` on ingest | rate limit or `DAILY_DOCUMENT_CAP` exceeded (see `Retry-After` header) |
| `410` on image download | image expired by the storage lifecycle |
| worker stuck / no OCR | check worker logs; models download on first run |

- API health: `GET /api/v1/health` (no auth).
- API docs / interactive console: `GET /docs` (Swagger) and `GET /redoc`.
