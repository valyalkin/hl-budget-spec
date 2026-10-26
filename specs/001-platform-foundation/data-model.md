# Data Model: Platform Foundation

**Feature**: `001-platform-foundation` | **Date**: 2026-10-25 | **Plan**: [plan.md](./plan.md)

Only the backend (`hl-budget-backend`) stores data (Constitution Principle II). The frontend holds a
session cookie and no data model of its own.

---

## User

A person who has signed in at least once. Every future budgeting record will reference it.

### Table `users` (Flyway `V1__create_users.sql`)

| Column | Type | Null | Default | Notes |
|---|---|---|---|---|
| `id` | `uuid` | no | `gen_random_uuid()` | Primary key. The opaque app identifier exposed as `id` in the API. Never a sequence |
| `auth0_subject` | `varchar(255)` | no | — | The verified token's `sub` (`auth0\|…`, `google-oauth2\|…`). The user's permanent key |
| `display_name` | `varchar(255)` | yes | — | From the `…/name` claim; refreshed on every profile call |
| `email` | `varchar(320)` | yes | — | From the `…/email` claim; refreshed on every profile call. **Not** an identity and **not** unique |
| `avatar_url` | `varchar(2048)` | yes | — | From the `…/picture` claim; refreshed on every profile call |
| `first_seen_at` | `timestamptz` | no | `now()` | Set on insert, never updated |
| `last_seen_at` | `timestamptz` | no | `now()` | Set on every profile call |
| `version` | `bigint` | no | `0` | Optimistic-lock column (`@Version`), per backend skill entity conventions |

Constraints and indexes:

- `pk_users` primary key on `id`.
- `uq_users_auth0_subject` unique on `auth0_subject` — enforces FR-014 and backs the
  `ON CONFLICT` upsert (research R-004).
- `ck_users_auth0_subject_not_blank` check `length(trim(auth0_subject)) > 0`.

No index on `email`: nothing looks users up by email (accounts are never merged by email,
spec Assumptions).

### Domain model (`model/User.kt`)

Immutable, annotation-free: `id: UUID`, `subject: String`, `displayName: String?`,
`email: String?`, `avatarUrl: String?`, `firstSeenAt: Instant`, `lastSeenAt: Instant`.

A separate value type `VerifiedIdentity(subject, displayName?, email?, avatarUrl?)` is built in
`config/` from the validated `Jwt` and handed to the service. It is the only input the service
accepts for "who is calling" (FR-015, FR-017).

### Validation rules (applied when building `VerifiedIdentity` from the token)

| Rule | Source | On failure |
|---|---|---|
| `sub` present and non-blank | FR-015 | 401 `urn:problem:hl-budget:unauthenticated` (token is treated as invalid; no row written) |
| `name` claim: trimmed, blank → null, truncated to 255 | Edge case "missing profile fields" | Stored as null |
| `email` claim: trimmed, blank → null, must contain `@`, max 320 | Edge case | Stored as null when invalid (logged at WARN without the value) |
| `picture` claim: must be an absolute `https` URL, max 2048 | Edge case; prevents `javascript:`/`http:` avatars | Stored as null |

### Lifecycle / state transitions

```text
(no record) ──first PUT /api/v1/users/me──▶ ACTIVE   [insert: first_seen_at = last_seen_at = now; 201]
ACTIVE      ──every PUT /api/v1/users/me──▶ ACTIVE   [update: last_seen_at = now; name/email/avatar = token values; 200]
```

No delete, deactivate or export in this feature (spec Assumptions, FR-028). Name or email changes
at Auth0 update the same row, because the row is keyed by `auth0_subject`, never by email.

### Relationships

None yet. Future features add tables with `user_id uuid not null references users(id)`, and
every query is filtered by the caller's `users.id` resolved from `sub` (Principle III).

---

## Session (frontend only, not persisted)

Owned by `@auth0/nextjs-auth0` in `hl-budget-frontend`; listed here because the spec names it as an
entity.

| Attribute | Value | Source |
|---|---|---|
| Storage | Encrypted (JWE, key from `AUTH0_SECRET`) HttpOnly, `Secure`, `SameSite=Lax` cookie `__session` | Principle III, FR-007 |
| Contents | ID-token claims (display only), access token, refresh token, expiry | SDK |
| Inactivity limit | 7 days (`inactivityDuration`), rolling | FR-008 |
| Absolute limit | 30 days (`absoluteDuration`) | FR-008 |
| End | `/auth/logout` clears the cookie and ends the Auth0 session | FR-009 |
| Server-side store | none (the backend holds no session) | Principle II |

---

## API view

The `UserProfile` response (see [contracts/openapi.yaml](./contracts/openapi.yaml)) maps 1:1 from
`User`: `id`, `displayName`, `email`, `avatarUrl`, `firstSeenAt`, `lastSeenAt`. `auth0_subject`
and `version` are never returned. Timestamps are ISO 8601 UTC (Principle VII).

### UI fallbacks (frontend, `app/lib/utils.ts`)

| Field missing | Greeting | Avatar |
|---|---|---|
| none | "Hello, {displayName}" | image from `avatarUrl` |
| `displayName` | "Hello, {email}" | as available |
| `displayName` and `email` | "Hello" | initials placeholder |
| `avatarUrl` | as available | initials from `displayName` or `email`, else a neutral icon |
| image fails to load | — | `onError` swaps to the initials placeholder (tiny client leaf) |
