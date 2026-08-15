# ADR 003 — JWT Authentication with Rotating Refresh Tokens + RBAC

**Status:** Accepted
**Date:** 2026-08-14

## Context

The family adds human users (web + mobile) on top of the existing machine-client API keys
(`document-processor` ADR-008). Humans need sessions with login, expiry, logout, and a permission
model (`ADMIN` vs `REVIEWER`), while machine clients keep `X-API-Key`.

## Decision

JWT Bearer for human users (web + mobile) with **rotating refresh tokens** and roles
`ADMIN`/`REVIEWER`, enforced by RBAC middleware. API keys remain for machine clients.

## Details

- `POST /auth/login` exchanges credentials for an access + refresh token pair.
- Access token is short-lived; `POST /auth/refresh` rotates the refresh token (old one revoked).
- `POST /auth/logout` revokes the current refresh token.
- `GET /auth/me` returns the authenticated user (`id`, `username`, `role`).
- Roles: `REVIEWER` (default) can list/review/export; `ADMIN` additionally manages API keys.
- Passwords hashed with argon2id; refresh tokens stored SHA-256-hashed (`refresh_tokens` table).
- Shared operations accept **either** `X-API-Key` or Bearer (contract `security` maps).
- `/health` and the auth endpoints remain unauthenticated.

## Alternatives Considered

| Option | Pros | Cons |
|--------|------|------|
| OAuth2/OIDC (external IdP) | Standards, federation | Adds an external dependency; overkill for a self-host v1 |
| Session cookies | Simple | Complicates the mobile client and CORS posture |
| **JWT + rotating refresh** | Stateless, mobile-friendly, revocable | Refresh-token rotation state to manage |

## Consequences

- `users` and `refresh_tokens` tables (plan §3).
- RBAC middleware checks `role` on review and API-key endpoints (403 on insufficient role).
- `user_id` ownership is added to `documents` (nullable for machine submissions).
