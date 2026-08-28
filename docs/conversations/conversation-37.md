# Conversation 37 — CI fix round: PAT permission + pnpm/Node pinning

> **Date:** 2026-08-28
> **Source session:** recovering the last session; diagnosing three failing PRs (hub contract-change
> dispatch, web CI, mobile CI)
> **Outcome:** all three root-caused and fixed. Pending HITL commit + PAT permission edit in GitHub.

## Summary

After the schemathesis/contract work, the four `develop → main` PRs were opened. Three failed in CI:

1. **Hub PR #1** (`contract-change`) — the `repository_dispatch` curl returned **403**. Root cause: the
   fine-grained `CROSS_REPO_PAT` was granted `Administration: Read and write`, but
   `POST /repos/{owner}/{repo}/dispatches` requires **`Contents: Read and write`**. The docs
   (`docs/ci-cd.md` §5) had the wrong permission.
2. **Web PR** and 3. **Mobile PR** — `pnpm/action-setup@v4` failed with "No pnpm version is specified":
   neither `packageManager` in `package.json` nor a `version:` input was present.

Fixed the docs, pinned pnpm via `packageManager`, and bumped CI Node 20 → 22.

## Decisions

| ID | Decision | Status |
|----|----------|--------|
| — | Fine-grained PAT for `repository_dispatch` requires **Contents: Read and write** (not Administration) | Applied (docs; token edit is HITL) |
| — | Pin pnpm via the `packageManager` field (`pnpm@11.22.0`) in each app's `package.json` — matches the dev machine and the v9.0 lockfile | Applied |
| — | CI Node 20 → **22** across web + mobile workflows (clears the Node 20 deprecation warning) | Applied |

## Deliverables

- **Hub:** `docs/ci-cd.md` §5 (Contents permission + `403` note), §2 (toolchain-pinning note),
  `docs/conversations/conversation-37.md`.
- **web:** `package.json` (+`packageManager`), `.github/workflows/ci.yml` + `contract-check.yml` (Node 22).
- **mobile:** `package.json` (+`packageManager`), `.github/workflows/ci.yml`, `mobile-e2e.yml`,
  `contract-check.yml` (Node 22).

## Verification

- `package.json` × 2 remain valid JSON; workflow YAML parses; local `pnpm --version` = `11.22.0` matches
  the pin; `pnpm-lock.yaml` still `lockfileVersion: '9.0'` (compatible, no reinstall needed).
- CI green expected after the HITL steps below.

## Notes / follow-ups

- **HITL (GitHub):** edit the fine-grained `CROSS_REPO_PAT` token → Repository permissions →
  **Contents: Read and write** (Administration can be removed), confirm the 3 consumer repos are
  selected, then **re-run** hub PR #1's failed check. The token value is unchanged, so the hub secret
  stays valid.
- **HITL (git):** commit + push each repo's changes on its `develop` branch (the PR head) to re-trigger
  CI: web and mobile carry the pnpm/Node fix; the hub carries the docs fix (a `docs/ci-cd.md` edit,
  which does **not** touch `docs/contracts/**` and so will not re-fire the contract-change dispatch).
- `document-processor` local `main` remains 5 commits behind `origin/main` (the merged PR) — pull when
  convenient; its stale local `develop` branch can be deleted (remote `origin/develop` was removed on
  merge).

## Transcript (condensed)

**user** — Recover the last session (4 repos, hub merged, others behind).

**assistant** — Reconstructed state; hub PR #1 still open, `document-processor` PR #1 merged, web/mobile
`develop` ahead by one.

**user** — Hub PR fails with `curl ... 403`.

**assistant** — Traced to the `CROSS_REPO_PAT` fine-grained permission (Administration ≠ Contents).

**user** — Web + mobile PRs fail: "No pnpm version is specified".

**assistant** — Root-caused: no `packageManager` / no `version` input; proposed pinning pnpm + bumping
Node.

**user** — Pin `pnpm@11.22.0`; also bump Node to 22. Build mode, proceed.

**assistant** — Applied all edits (package.json ×2, 5 workflow files, `ci-cd.md`); wrote this
conversation.
