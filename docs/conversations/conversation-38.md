# Conversation 38 — Mobile E2E pivot + JDK 21 fix

> **Date:** 2026-08-28
> **Source session:** recovering the CI failures; replacing the Maestro/emulator mobile E2E with a
> browser-based Playwright E2E
> **Outcome:** mobile E2E is now fast, self-contained, and CI-bound (no emulator); APK build kept as a
> separate optional job. Pending HITL commit.

## Summary

The mobile PR's Maestro E2E failed twice in CI. First the APK build hit `invalid source release: 21`
(Capacitor 7 needs JDK 21, CI had JDK 17). After fixing that, Maestro itself failed with
"Maestro Android driver did not start up in time" — the emulator booted in ~6.75 min because GitHub's
`ubuntu-latest` runners have no KVM (pure software emulation). Even a fast emulator would not help:
the Maestro flow does a real login against a live backend that CI never provisions.

Decision: drop Maestro and adopt a **browser-based Playwright E2E** against the Capacitor web bundle,
mirroring the web app — mock the API with `page.route` and the camera with an env-gated mock gateway.
The native APK build is retained as a separate, non-blocking job.

## Decisions

| ID | Decision | Status |
|----|----------|--------|
| — | Replace Maestro/emulator E2E with Playwright against the Capacitor web bundle (`VITE_MOCK_CAPTURE=1` + `page.route` mocks) | Applied |
| — | Camera mock via an env-gated `resolveCameraGateway()` (canvas-generated 1920×1080 JPEG) — no production behavior change (defaults to `capacitorCameraGateway`) | Applied |
| — | Native APK build kept as a separate optional job (`continue-on-error: true`, JDK 21) | Applied |
| — | Capacitor 7 requires JDK 21 (was 17) | Applied |

## Deliverables

- **`document-processor-mobile`:**
  - `src/capture/camera.ts` — `mockCameraGateway` + `resolveCameraGateway()`.
  - `src/App.tsx` — capture uses `resolveCameraGateway()`.
  - `playwright.config.ts` (new) — `webServer.env.VITE_MOCK_CAPTURE=1`.
  - `e2e/app.spec.ts` (new) — `login → capture → sync` with API mocks.
  - `package.json` — `@playwright/test` + `test:e2e` script.
  - `.gitignore` — `test-results/`, `playwright-report/`.
  - `.github/workflows/mobile-e2e.yml` — Playwright job + optional `android-build` job.
  - Removed `e2e/maestro/capture-sync.yaml`.
- **`document-processor-orchestration`:**
  - `docs/ci-cd.md` §2 — E2E description (mobile → Playwright) + toolchain note (JDK 21 → APK job).
  - `docs/conversations/conversation-38.md`.

## Verification

- Mobile `typecheck`/`lint`/`test` unaffected (tsconfig excludes `e2e/**` and `playwright.config.ts`,
  matching the web app; ESLint lints them, same as web).
- CI path mirrored from the web app (which already passes): `playwright install --with-deps chromium`
  → `playwright test`.

## Notes / follow-ups

- **Native APK build** is optional/non-blocking (`continue-on-error: true`); if it ever fails it shows
  a warning but won't block the PR. HITL asked for instructions/clarification on this job — pending.
- The Maestro flow (and its `reviewer`/`s3cret` + live-backend assumption) is retired; the browser E2E
  mocks both, so it no longer needs a running backend.
- Local dev: `pnpm test:e2e` runs the browser E2E; the mocked camera is enabled automatically by the
  Playwright webServer env.

## Transcript (condensed)

**user** — Mobile PR now fails at "Build debug APK" (`invalid source release: 21`).

**assistant** — Capacitor 7 needs JDK 21; bumped `setup-java` 17 → 21 in `mobile-e2e.yml`.

**user** — Another failure: "Run Maestro flows" (driver did not start in time).

**assistant** — No KVM on `ubuntu-latest` → software emulation (~6.75 min boot) → Maestro timeout; and
the flow needs a live backend CI doesn't provide. Proposed replacing Maestro with a browser Playwright
E2E (mocked camera + API), mirroring the web app.

**user** — Do the browser E2E; keep the APK build as a separate optional job.

**assistant** — Implemented the pivot + optional APK job; updated docs and recorded this conversation.
