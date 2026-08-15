# ADR 005 — Public-Exposure Cost & Abuse Governance

**Status:** Accepted
**Date:** 2026-08-14

## Context

The repos are public (portfolio) and the environment is paid from day one, so a public API must not
be drivable into unbounded cost. This extends `document-processor` ADR-008 (which covered per-key rate
limiting) into a complete guardrail set (decision D12).

## Decision

Bound cost by construction: auth-gated ingestion, rate limits, a global daily document cap plus
per-key quotas, storage lifecycle, and a budget kill-switch — all enforced before OCR/ingest work
runs.

## Details

| Guardrail | Mechanism |
|-----------|-----------|
| Auth-gated ingestion | OCR/upload requires JWT or API key (ADR 003) — no anonymous OCR |
| Edge protection | Cloudflare free in front of the VPS (DDoS/edge rate-limit) |
| Rate limiting | per-key/IP limits on ingest + image download (60 req/min baseline) |
| Daily document cap | 100 docs/day global + per-key quota, enforced before worker dequeues |
| Storage lifecycle | retention/archive; presigned URLs only (no public bucket) |
| Budget kill-switch | spend alerts on offsite backup; queue halts above the daily cap |
| Secrets hygiene | `.env` gitignored + `.env.example`; secrets via env/CI only |

## Alternatives Considered

| Option | Pros | Cons |
|--------|------|------|
| No cap, rely on rate limits only | Simpler | OCR is CPU/cost-heavy; rate alone doesn't bound daily spend |
| Free-tier hosting | Zero cost | Public API abuse + no data-residency control |
| **Quota + rate + lifecycle + kill-switch** | Bounds worst-case cost deterministically | Slightly more state to maintain |

## Consequences

- `daily_usage` table tracks global + per-key daily counts (plan §3).
- Ingest returns 429 with `Retry-After` when the cap or rate limit is exceeded.
- Worker checks the cap before dequeuing OCR work (fail-fast, cost-first).
