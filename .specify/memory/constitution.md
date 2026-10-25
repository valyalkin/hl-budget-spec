# HL Budget Constitution

## Core Principles

### I. Spec-Driven, Foundation-First Delivery

- Every feature MUST start as a Spec Kit specification (`/speckit-specify`), followed by a plan
  and tasks, before code is written.
- The first feature MUST deliver the platform foundation only: a running backend, a running
  frontend, and Auth0 login wired end to end. It MUST NOT contain budgeting business logic.
- Each later feature MUST be an independently deployable, user-visible increment that builds on
  a working foundation.

Rationale: proving the skeleton (build, auth, deploy) first keeps later business features small
and stops infrastructure problems from hiding inside domain work.

### II. Separate Services, Contract-First API

- The system consists of separately deployable services, one service per repository: a
  **frontend** (Next.js App Router, TypeScript) and a **backend** (Kotlin, Spring Boot).
- The backend owns all data, business rules and the HTTP contract. It renders no UI and holds no
  session.
- The frontend renders UI and calls the backend from the server. It MUST NOT contain a database
  client, ORM, SQL, or business rules, and it is not a proxy/aggregation BFF.
- The backend's published OpenAPI document is the single contract. Frontend API types MUST be
  generated from it, never hand-written.
- Every backend error MUST be an RFC 9457 `application/problem+json` response with a stable
  `type` URN and a `traceId`; a 5xx MUST NOT leak internals.
- The shape of the contract (versioning, resources, methods, payloads) is governed by
  Principle VII.

Rationale: a generated contract prevents silent drift between services, and a single owner for
data keeps rules in one place.

### III. Secure by Default (Auth0)

- Authentication MUST use Auth0. The frontend keeps the session in an encrypted HttpOnly cookie;
  access tokens MUST stay server-side and never reach browser JavaScript.
- The backend MUST validate the Auth0 JWT (issuer, audience, signature) on every request except
  the health endpoints, and MUST trust nothing else from the caller.
- Every piece of user data MUST be scoped to the authenticated user (Auth0 `sub`); a request MUST
  NOT be able to read or change another user's data.
- Secrets MUST arrive only as environment variables synced from Vault. No secret may be committed,
  baked into an image, passed as a build arg, written to logs, or placed in Helm values.
- Authenticated routes MUST be excluded from indexing (`robots: noindex`, not in the sitemap).

Rationale: this is personal financial data reachable over the network; token custody and
per-user scoping are the minimum bar.

### IV. Tests and Quality Gates (NON-NEGOTIABLE)

- Tests ship with the code. A change without tests is not done.
- Frontend: TypeScript `strict`, no `any`/`@ts-ignore`; unit tests for components, utils and data
  functions; a Playwright spec for every new user-visible flow. Gate:
  `pnpm typecheck && pnpm lint && pnpm test`.
- Backend: Kotlin with warnings as errors; unit and integration tests against real Postgres via
  Testcontainers (no H2); coverage floor of 80% line / 75% branch overall and 75% line per
  package. Gate: `./gradlew check`.
- The gate for each touched service MUST pass locally and in CI before work is reported finished
  or merged.

Rationale: a single developer has no reviewer safety net; automated gates are the reviewer.

### V. Homelab-Ready Operations

Every service MUST satisfy the homelab service contract from day one:

- Container: multi-stage Dockerfile, pinned base images (never `latest`), runs as UID/GID 10001
  non-root, read-only root filesystem with writes only to declared tmp dirs, no capabilities.
- HTTP: one port above 1024 (8080 by default, honouring `PORT`); `GET /healthz` (liveness, no
  dependency checks) and `GET /readyz` (fails while a required dependency such as Postgres is
  unreachable).
- When frontend and backend share a hostname, the frontend serves `/` and the backend serves its
  routes under `/api`.
- Configuration via environment variables only; `.env.example` documents them with no real
  values. Logs go to stdout/stderr. Clean shutdown on SIGTERM within 30 seconds.
- Database schema is owned by Flyway migrations (`ddl-auto: validate`); applied migrations are
  immutable.
- Deployment flows through the release workflow → ghcr.io image → hl-helm PR → Argo CD. No manual
  `kubectl apply` for application workloads.

Rationale: meeting the contract while building avoids rework at onboarding and keeps Argo CD's
health status truthful.

### VI. Simplicity at Personal Scale

- Design for a small number of users on a single homelab cluster. Do not add queues, caches,
  extra services or multi-region concerns until a spec shows a concrete need.
- Any added complexity (a new service, dependency or infrastructure component) MUST be justified
  in the plan's complexity tracking.

Rationale: every moving part is something one person has to operate.

### VII. Versioned, Consistent API Design

**Versioning**

- Every business endpoint MUST carry a major version in its path: `/api/v{N}/...` (starting at
  `/api/v1`). Only the operational endpoints (`/healthz`, `/readyz`) and the OpenAPI/Swagger
  documents are unversioned.
- Within a version, changes MUST be backward compatible: adding optional request fields, new
  response fields, new endpoints or new optional query parameters is allowed.
- A breaking change (removing or renaming a field or endpoint, changing a type, making an
  optional field required, changing status codes or error `type` URNs) MUST ship as a new major
  version (`/api/v{N+1}`). The previous version MUST keep working until the frontend no longer
  calls it.
- A deprecated version or endpoint MUST be marked `deprecated` in OpenAPI and send `Deprecation`
  and `Sunset` response headers.

**Resources and methods**

- URLs name resources, not actions: plural nouns in kebab-case (`/api/v1/budget-categories/{id}`),
  nested at most one level (`/api/v1/budgets/{id}/transactions`). No verbs in paths; an
  operation that is not CRUD is modelled as a sub-resource (`POST /transactions/{id}/reversal`).
- HTTP methods MUST follow their semantics: `GET` is safe and never changes state; `PUT` and
  `DELETE` are idempotent; `PATCH` is a partial update; `POST` creates.
- Status codes MUST be precise: `200` read/update, `201` create with a `Location` header, `204`
  no body, `400` malformed request, `401` missing/invalid token, `403` authenticated but not
  allowed, `404` not found (also for another user's resource, so existence is not leaked),
  `409` conflict, `422` validation or business-rule failure.

**Payloads**

- JSON with `camelCase` field names. Timestamps are ISO 8601 in UTC (`2026-10-25T09:15:00Z`),
  calendar dates are `YYYY-MM-DD`, identifiers are opaque strings (UUIDs), never database
  sequence numbers.
- Money is an object with an exact amount and an ISO 4217 currency, e.g.
  `{ "amount": "12.50", "currency": "EUR" }` (amount as a decimal string, never a JSON float).
- Enums are `UPPER_SNAKE_CASE` strings, documented in OpenAPI.
- Collection endpoints MUST be paginated (`page`/`size` or cursor), return the items in a named
  field with pagination metadata, and cap `size`. Filtering and sorting use query parameters
  (`?from=2026-10-01&sort=date,desc`).
- Request bodies MUST be validated at the boundary; every invalid field is reported in the
  problem response's `errors` array with a JSON Pointer.
- Endpoints that create money-moving records SHOULD accept an `Idempotency-Key` header so a
  retried request does not create duplicates.

**Documentation**

- Every endpoint, parameter, request and response (including error responses) MUST be described
  in the OpenAPI document with examples. An undocumented endpoint is a contract violation.

Rationale: path versioning keeps the generated frontend client stable while the API evolves,
and consistent conventions make every new endpoint predictable for both the frontend and tests.

## Technology & Platform Constraints

- **Frontend**: Next.js App Router, TypeScript, Tailwind, pnpm, ESLint (incl. `jsx-a11y`) +
  Prettier, Vitest/Jest + Playwright, Auth0 Next.js SDK.
- **Backend**: Kotlin, Spring Boot 4.x, Java 21, Gradle Kotlin DSL, JPA/Hibernate, Flyway,
  PostgreSQL, springdoc OpenAPI, ktlint + detekt + Kover, Testcontainers + WireMock.
- **Platform**: homelab Kubernetes via hl-helm charts, Argo CD, Traefik ingress (TLS terminated
  at Traefik), Vault for secrets, platform-provisioned Postgres.
- **Money**: monetary amounts MUST NOT be stored or computed as floating point. Use integer minor
  units or `BigDecimal`/`NUMERIC` with an explicit currency.
- Deviating from this stack is a constitutional amendment, not a plan decision.

## Development Workflow & Quality Gates

- Work happens on a feature branch per spec; `main` is always deployable.
- Order per feature: specify → clarify (if needed) → plan → tasks → implement → verify gates →
  merge → deploy via hl-helm.
- Backend contract changes land first; the frontend regenerates its API types in the same feature.
- Every contract change MUST state in its plan whether it is backward compatible or breaking;
  a breaking change requires a new API major version (Principle VII).
- Before a feature is considered done: all gates green, the service builds and runs under the
  hardened container constraints, and the deployed service reports healthy in Argo CD.
- Commits follow Conventional Commits (`feat:`, `fix:`, `docs:`, `chore:` …).

## Governance

- This constitution supersedes other practices. Plans MUST include a Constitution Check against
  these principles, and any violation MUST be justified in the plan's complexity tracking or the
  plan changes.
- Amendments are made through `/speckit-constitution`, recorded in the Sync Impact Report, and
  committed with a message describing the version change.
- Versioning follows semantic versioning: MAJOR for removing or redefining a principle, MINOR for
  a new principle or section or materially expanded guidance, PATCH for clarifications.
- Compliance is reviewed at each `/speckit-plan` and `/speckit-analyze` run and before merging a
  feature. Runtime guidance lives in the `frontend-developer`, `backend-developer` and `homelab:*`
  skills; where they conflict with this document, this document wins.

**Version**: 1.1.1 | **Ratified**: 2026-10-25 | **Last Amended**: 2026-10-25
