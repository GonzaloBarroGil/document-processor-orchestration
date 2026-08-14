# Conversation 5 — Spec phase: product spec (draft)

> **Date:** 2026-08-13
> **Source session:** continuation of `ses_008a8fee4ffeTqanPcqXgatHFi` (Spec phase)
> **Outcome:** drafted `docs/spec/product.md` v1.0 — **approved by HITL**.

## Summary

Completed the Spec phase by drafting the cross-component product spec (Gherkin). It promotes
the backend features F1–F9 unchanged and adds the family features: JWT auth (D9), document
browsing, review queue + manual fix, export, API-key administration, dashboard, mobile capture
with offline queue/dead-letter, and the D12 cost controls (daily quota).

## Decisions

| ID | Decision | Status |
|----|----------|--------|
| — | Product spec lives in `docs/spec/product.md`; backend F1–F9 promoted from document-processor | Applied |

## Transcript (condensed)

**user** — Update related status and then I'll commit changes.

**assistant** — Marked `docs/contracts/openapi.yaml` as "Approved v1.0" (x-status), updated
feasibility §13 and conversation-4.

**user** — Changes committed and pushed.

**assistant** — Drafted `docs/spec/product.md` (Gherkin): cross-component journeys, web app
features (auth, browsing, review, export, API-key admin, dashboard), mobile features (capture,
offline queue/dead-letter), and cost controls. Updated README and feasibility §13.
