# Implementation Plan: Platform Foundation

**Branch**: `001-platform-foundation` | **Date**: 2026-10-25 | **Spec**: [spec.md](./spec.md)

**Input**: Feature specification from `/specs/001-platform-foundation/spec.md`

## Summary

Deliver the HL Budget skeleton: a Next.js web app and a Kotlin/Spring Boot API, wired to Auth0
and deployed to the homelab on one private hostname (web at `/`, API at `/api`). A visitor signs
in through Auth0 Universal Login. The web app keeps the session in an encrypted HttpOnly cookie
and, on the server, calls `PUT /api/v1/users/me` with the access token. The API validates the
JWT (issuer, audience, signature, expiry), upserts exactly one `users` row per Auth0 `sub` in a
single `INSERT … ON CONFLICT` statement, and returns the profile, which the home page shows.
Failure paths (cancelled login, expired session, 401, API down, unknown URL) end in a friendly
page or a sign-in redirect. Both services meet the homelab service contract (UID 10001,
read-only root, `/healthz` + `/readyz`, port 8080) and ship through release workflow → ghcr →
hl-helm PR → Argo CD. No budgeting features.

## Technical Context

**Language/Version**: Backend Kotlin 2.3 on Java 21; frontend TypeScript 5.9 (strict) on Node 22

**Primary Dependencies**: Backend: Spring Boot 4.1 (web, security OAuth2 resource server,
data-jpa, validation, actuator, flyway), springdoc-openapi 3.1, PostgreSQL JDBC. Frontend:
Next.js 16 (App Router, Cache Components, React Compiler), React 19, `@auth0/nextjs-auth0` 4.32,
Tailwind 4, Zod 4, openapi-typescript 7

**Storage**: PostgreSQL 17 (platform CloudNativePG `17.7`), one table `users`, schema owned by
Flyway (`ddl-auto: validate`)

**Testing**: Backend: JUnit 5, MockK, AssertJ, Testcontainers Postgres 17, WireMock (JWKS),
Kover gates 80/75 + 75 per package; gate `./gradlew check`. Frontend: Vitest + RTL + MSW,
Playwright against an Auth0 test tenant; gate `pnpm typecheck && pnpm lint && pnpm test`

**Target Platform**: Single-node k3s homelab (Traefik TLS ingress, Vault + VSO, Argo CD);
images for the node's architecture (read at onboarding, research R-015)

**Project Type**: Web application, two services in two repositories (frontend + backend API)

**Performance Goals**: Home page with profile < 2 s on the home network (SC-005); sign-in
journey < 30 s (SC-001); readiness flips within 30 s (SC-007)

**Constraints**: Private network/VPN only; tokens never reach browser JS; session 7 d
inactivity / 30 d absolute; hardened containers (UID 10001, read-only root, no capabilities);
SIGTERM exit < 30 s; no secrets in Git, images, Helm values or logs

**Scale/Scope**: One to a handful of users, 1 replica per service, 1 API operation, 3 pages

All unknowns are resolved in [research.md](./research.md) (R-001 … R-016). No
`NEEDS CLARIFICATION` remains.

## Skills that drive the implementation

The user's development method is skill-driven. Every implementation task in `tasks.md` names the
skill that owns it, and that skill's definition of done is part of the task's done.

| Skill | Owns | Used in |
|---|---|---|
| `backend-developer` | Everything in `hl-budget-backend`: package layout (`config/ controller/ dto/ model/ repository/ error/ service/`), problem+json error package, Auth0 resource server, typed config, Flyway, Testcontainers/WireMock tests, `contracts/api.yaml` export, `./gradlew check` | Backend setup, US1, US2, US5 (health), US6 |
| `frontend-developer` | Everything in `hl-budget-frontend`: `app/` + `app/ui/` + `app/lib/` layout, `proxy.ts` gating, `app/lib/{env,api,auth0,auth,data}.ts`, generated API types, metadata/robots, error/not-found boundaries, Vitest/MSW/Playwright, `pnpm` gates | Frontend setup, US1–US4, US6 |
| `homelab:hl-containerize` | Each repo's Dockerfile + `.dockerignore`, proof under `--read-only --user 10001:10001 --cap-drop ALL`, list of tmpDirs | US5, before onboarding |
| `homelab:hl-onboard-service` | `homelab.yaml`, `.github/workflows/release.yaml` (template, unchanged), GitHub repo, first image, `charts/apps/hl-budget` in hl-helm (api first, then web on the same host), Postgres DB entry, onboarding PR and operator checklist | US5 |
| `homelab:hl-service-config` | Any later chart change: TLS secret, env var, memory, probes, host | US5 follow-ups, R-012 TLS if not in the onboarding PR |
| `homelab:hl-deploy-status` | Read-only verification after each hl-helm merge: workflow → PR → Argo CD → pods → Vault sync | US5 acceptance, SC-004 |

Where a skill and the constitution disagree, the constitution wins (Governance). The three
reconciliations made in this plan are research R-006 (health paths vs `/actuator`), R-007
(datasource env vs homelab `POSTGRES_*`) and R-007's Postgres 17 pin (vs the skill example's 18).

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

| Principle | Requirement | Plan | Status |
|---|---|---|---|
| I. Spec-driven, foundation-first | Spec → plan → tasks before code; foundation only, no budgeting | Spec ratified and clarified; scope = sign-in, profile, health, deploy. FR-028 excluded list honoured | PASS |
| II. Separate services, contract-first | Next.js + Spring Boot, one repo each; backend owns data and contract; frontend has no DB/ORM/rules, not a BFF; types generated from OpenAPI; RFC 9457 errors with `type` URN + `traceId` | `hl-budget-frontend` + `hl-budget-backend` (R-001); frontend calls the API from the server via `app/lib/api.ts`; `pnpm generate:api` from `contracts/api.yaml`; error package per backend skill (R-014) | PASS |
| III. Secure by default | Auth0; session in encrypted HttpOnly cookie; tokens server-side; JWT iss/aud/signature validated on every request except health; data scoped to `sub`; secrets only from Vault; authed routes `noindex` | `@auth0/nextjs-auth0` v4, `/auth/access-token` disabled (web-routes.md); explicit `aud` validator (R-005); identity only from token, no id input (R-002); Vault paths only (deployment.md); `/home` `noindex`, not in sitemap | PASS |
| IV. Tests and quality gates | Tests with code; TS strict, no `any`; Playwright per flow; Kotlin warnings as errors; Testcontainers (no H2); 80/75 + 75 per package; gates green locally and in CI | Test map in R-016; Playwright specs per story; `./gradlew check` and pnpm gates in each repo's CI | PASS |
| V. Homelab-ready operations | Multi-stage pinned images, UID 10001, read-only root, no caps; port 8080/`PORT`; `/healthz` + `/readyz` (readiness fails without Postgres); web `/`, API `/api`; env-only config + `.env.example`; stdout logs; SIGTERM < 30 s; Flyway validate; release → ghcr → hl-helm → Argo CD | R-015 containers proven by `hl-containerize`; R-008 health on both; R-006 `/api` routes; deployment.md env tables; graceful shutdown; Flyway V1; `hl-onboard-service` pipeline | PASS |
| VI. Simplicity at personal scale | No queues/caches/extra services without need; additions justified | Two services, one table, one replica each, stateless session (no session store), no cert-manager (R-012), no observability vendor (R-014) | PASS |
| VII. Versioned, consistent API | `/api/v{N}`; resources not verbs; method semantics; precise status codes; camelCase, ISO UTC, UUID ids; documented with examples incl. errors | `PUT /api/v1/users/me` (idempotent upsert; GET would change state, R-002); 201 + `Location` on create, 200 otherwise, 401 problem; `UserProfile` camelCase, `uuid` id, UTC timestamps; full OpenAPI with examples ([contracts/openapi.yaml](./contracts/openapi.yaml)). Contract change: **new API, v1, no compatibility concern** | PASS |
| Tech & platform constraints | Stack as listed; no float money | Exactly the listed stack; no money fields in this feature | PASS |

**Post-design re-check (after Phase 1)**: PASS. The design adds no service, queue, cache or
store beyond the constitution's stack. Two items depend on the platform rather than on this
feature, and are tracked as operator steps rather than violations:

- **Trusted TLS certificate** (SC-004): hl-helm currently serves a self-signed default. Solved
  inside the existing chart contract with `ingress.tls.secretName` + a Vault-synced
  `kubernetes.io/tls` secret (R-012). Automatic renewal (cert-manager) is a platform follow-up.
- **Auth0 tenant configuration** (sign-up off, Google, refresh-token lifetimes, Post-Login
  Action): versioned as source in `hl-budget-backend/auth0/` and applied by the operator
  (quickstart §1).

## Project Structure

### Documentation (this feature)

```text
specs/001-platform-foundation/
├── plan.md              # This file
├── research.md          # Phase 0: decisions R-001 … R-016
├── data-model.md        # Phase 1: users table, VerifiedIdentity, session, UI fallbacks
├── quickstart.md        # Phase 1: local, gates, container, deploy and journey checks
├── contracts/
│   ├── openapi.yaml     # PUT /api/v1/users/me, /healthz, /readyz, Problem
│   ├── web-routes.md    # Pages, SDK routes, gating, backend call, security headers
│   └── deployment.md    # homelab.yaml, hl-helm chart shape, env vars, Vault paths
├── checklists/requirements.md
└── tasks.md             # Phase 2 (/speckit-tasks; not created here)
```

### Source Code

Two service repositories, each checked out as a git worktree on branch
`001-platform-foundation` inside this repo (`hl-budget-spec/hl-budget-backend/`,
`hl-budget-spec/hl-budget-frontend/`; gitignored here, main clones in `~/Documents/Projects/`),
plus the existing `~/Documents/Projects/hl-helm`. Layouts follow the skills' reference projects (`backend-developer` `assets/example/`, `frontend-developer` `assets/example/`).

```text
hl-budget-backend/                                  # backend-developer skill
├── build.gradle.kts, settings.gradle.kts, gradle/libs.versions.toml
├── detekt.yml, .editorconfig, compose.yaml (postgres:17.7-alpine, no fixed host port)
├── Dockerfile, .dockerignore                   # hl-containerize
├── homelab.yaml, .github/workflows/release.yaml   # hl-onboard-service
├── .github/workflows/ci.yaml                   # ./gradlew check
├── .env.example, README.md
├── auth0/post-login-profile-claims.js          # Auth0 Action source (R-003)
├── contracts/api.yaml                          # exported by OpenApiExportIT, consumed by web
└── src/
    ├── main/kotlin/<pkg>/hlbudget/
    │   ├── HlBudgetApplication.kt
    │   ├── config/      SecurityConfig, HlBudgetProperties, RequestIdFilter, JacksonConfig,
    │   │                OpenApiConfig, VerifiedIdentityResolver (Jwt → VerifiedIdentity)
    │   ├── controller/  UserController (PUT /api/v1/users/me)
    │   ├── dto/         UserProfileResponse + mappings
    │   ├── model/       User, VerifiedIdentity
    │   ├── repository/  UserEntity, UserRepository (native upsert), UserMappings
    │   ├── service/     UserService (upsert, created flag)
    │   └── error/       ProblemCodes, BusinessException tree, SystemException, FieldViolation,
    │                    ViolationExtractor, ProblemDetailFactory, GlobalExceptionHandler,
    │                    ProblemErrorController
    ├── main/resources/  application.yaml, application-local.yaml,
    │                    db/migration/V1__create_users.sql
    └── test/kotlin/<pkg>/hlbudget/
        ├── support/     AbstractIntegrationTest, JwtTestTokens
        ├── service/     UserServiceTest
        ├── controller/  UserControllerIT, UserProvisioningIT
        ├── config/      SecurityIT, OpenApiExportIT, ReadinessIT, HealthIT
        ├── error/       GlobalExceptionHandlerIT
        └── repository/  SchemaValidationIT

hl-budget-frontend/                                  # frontend-developer skill
├── package.json, pnpm-lock.yaml, tsconfig.json, next.config.ts (output: 'standalone')
├── eslint.config.mjs, .prettierrc, vitest.config.ts, vitest.setup.ts, playwright.config.ts
├── Dockerfile, .dockerignore                   # hl-containerize
├── homelab.yaml, .github/workflows/release.yaml   # hl-onboard-service
├── .github/workflows/ci.yaml                   # pnpm gates, build, generate:api diff, e2e
├── .env.example, README.md
├── contracts/api.yaml                          # copied from hl-budget-backend
├── proxy.ts                                    # Auth0 mount, /home gating, no-store
├── instrumentation.ts                          # onRequestError → stdout, no headers
├── app/
│   ├── layout.tsx, page.tsx (landing), not-found.tsx, global-error.tsx
│   ├── robots.ts, sitemap.ts, icon.png
│   ├── healthz/route.ts, readyz/route.ts
│   ├── home/        page.tsx, loading.tsx, error.tsx
│   ├── ui/
│   │   ├── landing/ sign-in-notice.tsx, landing-actions.tsx
│   │   ├── home/    profile-card.tsx, avatar.tsx (client leaf for onError), sign-out-link.tsx
│   │   ├── skeletons.tsx, fonts.ts
│   └── lib/
│       ├── env.ts, api.ts, auth0.ts, auth.ts, data.ts, definitions.ts, utils.ts,
│       ├── safe-return-to.ts, observability.ts
│       └── api-schema.d.ts                     # generated, committed
├── app/**/*.test.ts(x)                         # Vitest + RTL + MSW, colocated
└── e2e/ sign-in.spec.ts, protected-routes.spec.ts, sign-out.spec.ts, failures.spec.ts,
         not-found.spec.ts

hl-helm/ (existing)                             # hl-onboard-service, hl-service-config
├── charts/apps/hl-budget/  Chart.yaml, Chart.lock, values.yaml, values.schema.json,
│                           README.md, ci/default-values.yaml,
│                           templates/{namespace,backend,frontend}.yaml
└── bootstrap/values/postgres.yaml              # + hl-budget
```

**Structure Decision**: Two service repositories as required by Constitution Principle II and
the homelab's one-service-per-repo rule (`homelab.yaml`), plus a chart in the existing hl-helm
repo. This spec repo hosts their worktrees but commits no code (research R-001). Each feature
gets its own branch and worktree in both service repos; merge there, then remove the worktree. Inside each service repo the layout is the one its developer
skill prescribes. No top-level `src/`, `components/` or `services/` in the web app; no `util/` or
`mapper/` packages in the API.

## Delivery order

1. **API skeleton** (`backend-developer`): Gradle + gates, error package, request-id filter,
   security config, typed properties, health on `/healthz` `/readyz`, Flyway V1, upsert,
   `PUT /api/v1/users/me`, tests, `contracts/api.yaml` export. Contract lands first
   (Development Workflow).
2. **Web skeleton** (`frontend-developer`): tooling + gates, env, api, auth0 client + proxy,
   landing, `/home`, error/not-found, health routes, `generate:api` from step 1's contract,
   unit tests, Playwright specs.
3. **Containers** (`hl-containerize`) in both repos.
4. **Onboard backend** (`hl-onboard-service`): new app `hl-budget`, Postgres, `/api` on `<host>`.
5. **Onboard frontend** (`hl-onboard-service`): `frontend` block on the same `<host>` at `/`, TLS secret
   (R-012) — or a follow-up via `hl-service-config`.
6. **Verify** (`hl-deploy-status`) and run the quickstart deployed and journey checks.

## Complexity Tracking

No constitution violations. Nothing to justify.
