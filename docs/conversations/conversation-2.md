# Conversation 2 — Hosting + public-exposure governance

> **Date:** 2026-08-13
> **Source session:** continuation of `ses_008a8fee4ffeTqanPcqXgatHFi` (Hosting follow-up)
> **Outcome:** decisions D11 (Hosting) and D12 (cost & abuse guardrails) recorded; `docs/feasibility.md` advanced to v0.3.

## Summary

Resolved the last structural open question — Hosting — and, prompted by the fact that the
repos will be public (portfolio) and the environment is paid from day one, added explicit
cost & abuse guardrails. All open questions are now resolved except Manual review, License,
and On-device OCR (WASM).

## Decisions

| ID | Decision | Status |
|----|----------|--------|
| D11 | Hosting: self-host VPS (docker-compose) as v1 default; provider-agnostic (4 vCPU / 8 GB, Hetzner suggested); managed services as the documented migration path | Accepted |
| D12 | Public exposure & cost governance: Cloudflare free edge, auth-gated ingestion, rate limits, 100 docs/day global cap + per-key quotas, storage lifecycle + signed URLs, budget kill-switch, secrets hygiene | Accepted |

## Context collected from HITL

- Data residency: **international managed is fine** (no hard AR residency block).
- Ops capacity: **self-hosting preferred**, with AI guidance; wants the environment to be
  scalable and open to a managed future.
- Budget: **minimize cost**.
- Volume: **100–1000 docs/day**.
- Repos will be **public (portfolio)**, so the paid environment needs security/cost controls.

## Transcript (condensed)

**user** — Changes committed and pushed to new remote repo. Also created `document-processor-web`
and `document-processor-mobile` local git repos. Let's continue with Hosting.

**assistant** — (questions) Data residency / ops capacity / budget / volume.

**user** — International managed is fine; self-hosting sounds good (with AI guidance), still want an
evaluation of scalability and openness to a better managed environment; minimize cost; 100–1000/day.

**assistant** — Recommended D11 = self-host VPS default with a documented managed-migration path;
presented reference topology + scaling ladder.

**user** — Both questions made me think of additional issues: repos will be public (portfolio), and
since we won't start with a free environment, we should enforce some security level (don't want to
waste money), or restrict the daily number of processed documents.

**assistant** — Proposed D12 (public exposure & cost governance) guardrails table.

**user** — 100 docs/day cap; yes to Cloudflare free; provider-agnostic but keep Hetzner as a
suggested option.

**assistant** — Final consolidated plan.

**user** — I agree. Proceed.
