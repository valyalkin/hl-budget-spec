---

description: "Task list for 001-platform-foundation"
---

# Tasks: Platform Foundation

**Input**: Design documents from `/specs/001-platform-foundation/`

**Prerequisites**: plan.md, spec.md, research.md, data-model.md, contracts/ (openapi.yaml,
web-routes.md, deployment.md), quickstart.md

**Tests**: REQUIRED. Constitution Principle IV ("tests ship with the code") and research R-016 map
every success criterion to a test. Within each story, write the tests with the code and make sure
each new test fails before the code that satisfies it exists.

**Organization**: Tasks are grouped by user story so each story can be built and checked on its
own.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Can run in parallel (different files, no dependency on an unfinished task)
- **[Story]**: The user story the task belongs to (US1 … US6)
- Every task ends with `· skill: <name>`. Load that skill before starting the task; its
  definition of done is part of the task's done (plan.md "Skills that drive the implementation").
  Where a skill and the constitution disagree, the constitution wins (research R-006, R-007).

## Path Conventions

All paths are relative to the spec repo root `/Users/valyalkin/Documents/Projects/hl-budget-spec/`
unless they start with `~`.

- `hl-budget-backend/` — git worktree of `valyalkin/hl-budget-backend`, branch
  `001-platform-foundation`. Kotlin package `com.valyalkin.hlbudget`, so main code is under
  `hl-budget-backend/src/main/kotlin/com/valyalkin/hlbudget/` and tests under
  `hl-budget-backend/src/test/kotlin/com/valyalkin/hlbudget/`.
- `hl-budget-frontend/` — git worktree of `valyalkin/hl-budget-frontend`, branch
  `001-platform-foundation`.
- `~/Documents/Projects/hl-helm/` — the existing homelab chart repo.
- Commit inside the service repos (not in the spec repo, which ignores them), using Conventional
  Commits (`feat:`, `test:`, `chore:` …), after each task or logical group.
- Merging to `main` in a service repo triggers a release. Do it only after the repo's gate is
  green and the user agrees.

---

## Phase 1: Setup (Shared Infrastructure)

**Purpose**: Both service repos build, lint and test (empty) green locally and in CI.

- [X] T001 Check preconditions: `git -C hl-budget-backend branch --show-current` and `git -C hl-budget-frontend branch --show-current` both print `001-platform-foundation`; both worktrees are empty apart from `.git`; `~/Documents/Projects/hl-helm` exists on a clean `main`; Docker, JDK 21, Node 22 and pnpm 10 are installed (quickstart "Prerequisites"). Stop and report if any check fails · skill: none
- [X] T002 Scaffold the backend Gradle project from the skill's `assets/example/`: `hl-budget-backend/settings.gradle.kts` (rootProject `hl-budget-backend`), `hl-budget-backend/build.gradle.kts`, `hl-budget-backend/gradle/libs.versions.toml`, Gradle wrapper, and `hl-budget-backend/src/main/kotlin/com/valyalkin/hlbudget/HlBudgetApplication.kt` with `@ConfigurationPropertiesScan`. Versions: Kotlin 2.3, Java 21 toolchain, Spring Boot 4.1 (starters web, security, oauth2-resource-server, data-jpa, validation, actuator, flyway + `flyway-database-postgresql`), springdoc-openapi 3.1, PostgreSQL JDBC, `spring-boot-docker-compose` as `developmentOnly`; tests: JUnit 5, MockK, AssertJ, Testcontainers `postgresql`, `spring-boot-testcontainers`, WireMock. Kotlin `allWarningsAsErrors = true` · skill: backend-developer
- [X] T003 [P] Add backend lint and coverage gates: `hl-budget-backend/detekt.yml`, `hl-budget-backend/.editorconfig`, ktlint and detekt plugins, and Kover verify rules in `hl-budget-backend/build.gradle.kts` — overall line ≥ 80 %, branch ≥ 75 %, every package line ≥ 75 %; `./gradlew check` runs ktlint, detekt, tests and `koverVerify` · skill: backend-developer
- [X] T004 [P] Add backend base configuration: `hl-budget-backend/compose.yaml` (service `postgres`, image `postgres:17.7-alpine`, db/user/password `hl-budget`/`hl-budget`/`hl-budget`, **no fixed host port**), `hl-budget-backend/src/main/resources/application.yaml` with `server.port: ${PORT:8080}`, `server.shutdown: graceful`, `spring.lifecycle.timeout-per-shutdown-phase: 20s`, `spring.jpa.hibernate.ddl-auto: validate`, `spring.jpa.open-in-view: false`, Hikari `connection-timeout: 2000` and `validation-timeout: 1000`, springdoc `api-docs.path: /api/v3/api-docs` and `swagger-ui.path: /api/swagger-ui.html`, and **no `spring.datasource` block** (research R-007); `hl-budget-backend/src/main/resources/application-local.yaml` enabling docker-compose support · skill: backend-developer
- [X] T005 [P] Add `hl-budget-backend/.github/workflows/ci.yaml`: on pull request and push, JDK 21 + Gradle cache, run `./gradlew check` (Docker available for Testcontainers), plus a gitleaks job; and `hl-budget-backend/.gitignore` ignoring `build/`, `.gradle/`, `.env*` except `.env.example` · skill: backend-developer
- [X] T006 Scaffold the frontend from the skill's `assets/example/`: `hl-budget-frontend/package.json` (`packageManager: pnpm@10`, `engines.node: 22`; scripts `dev`, `build`, `start`, `typecheck` = `tsc --noEmit`, `lint`, `test` = `vitest run`, `e2e` = `playwright test`, `generate:api` = `openapi-typescript contracts/api.yaml -o app/lib/api-schema.d.ts`), dependencies Next.js 16, React 19, `@auth0/nextjs-auth0` 4.32.x, Tailwind 4, Zod 4, `server-only`; dev dependencies TypeScript 5.9, openapi-typescript 7, Vitest, RTL, MSW, Playwright; `hl-budget-frontend/tsconfig.json` (`strict: true`, path alias `@/*`); `hl-budget-frontend/next.config.ts` (`output: 'standalone'`, `reactCompiler: true`, `cacheComponents: true`); `hl-budget-frontend/app/globals.css` with Tailwind 4; commit `pnpm-lock.yaml` · skill: frontend-developer
- [X] T007 [P] Add frontend lint and format config: `hl-budget-frontend/eslint.config.mjs` (Next core-web-vitals, `jsx-a11y`, `@typescript-eslint/no-explicit-any: error`, ban `@ts-ignore`) and `hl-budget-frontend/.prettierrc` · skill: frontend-developer
- [X] T008 [P] Add frontend test config: `hl-budget-frontend/vitest.config.ts` (jsdom, colocated `app/**/*.test.ts(x)`, `@/` alias), `hl-budget-frontend/vitest.setup.ts` (jest-dom matchers, MSW `setupServer` started/reset/closed around each test, `onUnhandledRequest: 'error'`), `hl-budget-frontend/playwright.config.ts` (`baseURL` from `APP_BASE_URL`, default `http://localhost:3000`; specs in `e2e/`) · skill: frontend-developer
- [X] T009 [P] Add `hl-budget-frontend/.github/workflows/ci.yaml`: pnpm install `--frozen-lockfile`, `pnpm typecheck`, `pnpm lint`, `pnpm test`, `pnpm build`, `pnpm generate:api && git diff --exit-code app/lib/api-schema.d.ts`, gitleaks, and an `e2e` job using repository secrets for the Auth0 test tenant; and `hl-budget-frontend/.gitignore` ignoring `.next/`, `node_modules/`, `test-results/`, `playwright-report/`, `.env*` except `.env.example` · skill: frontend-developer

**Checkpoint**: `./gradlew check` and `pnpm typecheck && pnpm lint && pnpm test` pass on empty projects.

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: The backend's error, request-id, configuration and token-validation core, and the
frontend's env, Auth0 client, proxy and API client. Every user story uses these.

**⚠️ CRITICAL**: No user story work can begin until this phase is complete.

### Backend core

- [X] T010 Create the error package in `hl-budget-backend/src/main/kotlin/com/valyalkin/hlbudget/error/`: `ProblemCodes.kt` (URNs exactly `urn:problem:hl-budget:unauthenticated`, `access-denied`, `bad-request`, `not-found`, `conflict`, `unprocessable-entity`, `internal-error`, `service-unavailable`), `BusinessException.kt` tree, `SystemException.kt`, `FieldViolation.kt` (`pointer`, `field`, `code`, `message`), `ViolationExtractor.kt`, `ProblemDetailFactory.kt` (adds `instance`, `timestamp` ISO UTC, `traceId`; every 5xx gets the fixed detail "An unexpected error occurred"), `GlobalExceptionHandler.kt`, `ProblemErrorController.kt` (unknown paths and container errors → problem+json) — shape per contracts/openapi.yaml `Problem` (research R-014) · skill: backend-developer
- [X] T011 [P] Create `hl-budget-backend/src/main/kotlin/com/valyalkin/hlbudget/config/RequestIdFilter.kt` (servlet filter at order `-110`, reuses an incoming `X-Request-Id` or generates a 16-hex id, puts it in MDC as `traceId`, echoes it in the `X-Request-Id` response header) and `config/JacksonConfig.kt` (ignore unknown properties, ISO-8601 UTC timestamps, no numeric dates) · skill: backend-developer
- [X] T012 [P] Create `hl-budget-backend/src/main/kotlin/com/valyalkin/hlbudget/config/HlBudgetProperties.kt`: `@ConfigurationProperties("hlbudget")` + `@Validated`, nested `auth` with `issuer`, `audience`, `jwkSetUri`, `claimsNamespace`, each `@NotBlank` (bound from `HLBUDGET_AUTH_ISSUER`, `HLBUDGET_AUTH_AUDIENCE`, `HLBUDGET_AUTH_JWKSETURI`, `HLBUDGET_AUTH_CLAIMSNAMESPACE`); add the `hlbudget.auth.*` keys to `application.yaml` with no defaults so startup fails naming the missing one · skill: backend-developer
- [X] T013 Create `hl-budget-backend/src/main/kotlin/com/valyalkin/hlbudget/config/SecurityConfig.kt` (depends on T010–T012): stateless OAuth2 resource server; `NimbusJwtDecoder.withJwkSetUri(jwkSetUri)`; `DelegatingOAuth2TokenValidator` of issuer + timestamp validators **and** an explicit, null-safe audience validator that rejects a token whose `aud` is missing or does not contain `audience` (research R-005; Spring's default does not check `aud`); CSRF off; `permitAll` only for `/healthz`, `/readyz`, `/api/v3/api-docs/**`, `/api/swagger-ui/**`, `/api/swagger-ui.html`, `/error`; everything else authenticated; authentication entry point and access-denied handler write UTF-8 `application/problem+json` (`unauthenticated` 401 with `WWW-Authenticate: Bearer`, `access-denied` 403) through `ProblemDetailFactory`, carrying `traceId` · skill: backend-developer
- [X] T014 [P] Create `hl-budget-backend/src/main/kotlin/com/valyalkin/hlbudget/config/OpenApiConfig.kt`: info title "HL Budget API", version `1.0.0`, description from contracts/openapi.yaml, server `/`, `bearerAuth` HTTP bearer JWT scheme applied globally, `Problem` and `FieldViolation` schemas, tags `Users` and `Operations` · skill: backend-developer
- [X] T015 Create test support in `hl-budget-backend/src/test/kotlin/com/valyalkin/hlbudget/support/`: `AbstractIntegrationTest.kt` (`@SpringBootTest(webEnvironment = RANDOM_PORT)`, a shared `postgres:17.7-alpine` Testcontainer with `@ServiceConnection`, a WireMock server serving the test JWKS at `/.well-known/jwks.json`, `@DynamicPropertySource` for `hlbudget.auth.*`, a `jdbcTemplate` helper and a `users` table truncate between tests) and `JwtTestTokens.kt` (RSA key pair + JWKS JSON; builders for a valid token with `sub` and the three namespaced claims, and for expired, wrong issuer, wrong audience, missing `aud`, missing/blank `sub`, signed by an unknown key, and malformed) · skill: backend-developer
- [X] T016 Write `hl-budget-backend/src/test/kotlin/com/valyalkin/hlbudget/error/GlobalExceptionHandlerIT.kt`: an unknown path returns a problem+json with `traceId` equal to the `X-Request-Id` header; an unauthenticated request returns `urn:problem:hl-budget:unauthenticated`; a forced unexpected exception (test-only controller) returns 500 `internal-error` with detail "An unexpected error occurred" and no exception text; a supplied `X-Request-Id` is echoed · skill: backend-developer

### Frontend core

- [X] T017 [P] Create `hl-budget-frontend/app/lib/env.ts` (`import 'server-only'`; Zod 4 schema: `APP_BASE_URL` url, `API_BASE_URL` url, `API_TIMEOUT_MS` coerced positive int default `5000`, `AUTH0_DOMAIN`, `AUTH0_CLIENT_ID`, `AUTH0_CLIENT_SECRET`, `AUTH0_SECRET`, `AUTH0_AUDIENCE` non-empty strings; parse once and throw an error whose message names every missing or invalid variable, never its value) and `hl-budget-frontend/app/lib/env.test.ts` (valid env parses; each missing variable is named in the error; values are not printed) · skill: frontend-developer
- [X] T018 Create `hl-budget-frontend/app/lib/auth0.ts` (depends on T017): `Auth0Client` with domain, client id/secret, secret, `appBaseUrl` from env; `authorizationParameters: { scope: 'openid profile email offline_access', audience: AUTH0_AUDIENCE }`; `session: { rolling: true, inactivityDuration: 60*60*24*7, absoluteDuration: 60*60*24*30 }`; routes `/auth/login`, `/auth/callback`, `/auth/logout`; `enableAccessTokenEndpoint: false` and the profile endpoint disabled (web-routes.md); and `hl-budget-frontend/app/lib/auth.ts` (`server-only`; `getSession()`, `getAccessToken()` wrapping the SDK; nothing token-related is exported to client components) · skill: frontend-developer
- [X] T019 Create `hl-budget-frontend/proxy.ts` (depends on T018): run `auth0.middleware(request)`; for paths under `/auth/` return the SDK response; for `PROTECTED_PREFIXES = ['/home']` without a session redirect `302` to `/auth/login?returnTo=<encoded path+query>` with no page content; on protected pass-through return the SDK response (so the rolling cookie refreshes) with `Cache-Control: private, no-cache, no-store`; `config.matcher` excludes `_next/static`, `_next/image`, `favicon.ico`, `robots.txt`, `sitemap.xml`, `healthz`, `readyz` · skill: frontend-developer
- [X] T020 [P] Create `hl-budget-frontend/app/lib/api.ts` (`server-only`): `apiFetch(path, init)` joining `API_BASE_URL` + a path that must start with `/api/v1/`, with `AbortSignal.timeout(API_TIMEOUT_MS)` and `cache: 'no-store'`; `authedFetch` adding `Authorization: Bearer <access token>` from `getAccessToken()`; on non-2xx parse the problem body and throw `ApiError { status, type, traceId }`, with `UnauthenticatedError` for 401; timeouts and network failures throw `ApiUnavailableError`. Add `hl-budget-frontend/app/lib/definitions.ts` for these error classes and `hl-budget-frontend/app/lib/api.test.ts` (MSW: 2xx returns JSON; 401 → `UnauthenticatedError`; 500 problem → `ApiError` with `traceId`; timeout → `ApiUnavailableError`; bearer header sent) · skill: frontend-developer
- [X] T021 [P] Create `hl-budget-frontend/app/layout.tsx` (`<html lang="en">`, fonts from `app/ui/fonts.ts`, metadata `title: { default: 'HL Budget', template: '%s · HL Budget' }`, `metadataBase` from `APP_BASE_URL`), `hl-budget-frontend/app/ui/fonts.ts`, and security headers in `hl-budget-frontend/next.config.ts` `headers()` for all routes: `Content-Security-Policy` (`default-src 'self'`; `img-src 'self' https: data:`; scripts/styles per the skill's `tooling.md` "Security headers"), `X-Content-Type-Options: nosniff`, `Referrer-Policy: strict-origin-when-cross-origin`, `X-Frame-Options: DENY`, `Strict-Transport-Security: max-age=31536000` · skill: frontend-developer
- [X] T022 [P] Create `hl-budget-frontend/instrumentation.ts` (`onRequestError` logs one JSON line to stdout with path, method, digest and error message, **never request headers or cookies**) and `hl-budget-frontend/app/lib/observability.ts` (the shared logger it uses), with `hl-budget-frontend/app/lib/observability.test.ts` asserting headers are not logged · skill: frontend-developer

**Checkpoint**: `./gradlew check` green (including `GlobalExceptionHandlerIT`); frontend gate green. User stories can start.

---

## Phase 3: User Story 1 — Sign in and see my own profile (Priority: P1) 🎯 MVP

**Goal**: A visitor signs in through Auth0 and the home page shows their name, email and avatar as
returned by `PUT /api/v1/users/me`, which creates exactly one `users` row per Auth0 `sub`.

**Independent Test**: Signed out, click "Sign in", complete login, see the correct name, email and
avatar on `/home`. Sign in again: still one row for that `sub`, `last_seen_at` advanced
(quickstart L3–L5).

### Tests for User Story 1 ⚠️ (write first, see them fail)

- [X] T023 [P] [US1] Write `hl-budget-backend/src/test/kotlin/com/valyalkin/hlbudget/repository/SchemaValidationIT.kt`: context starts with `ddl-auto: validate` against Flyway V1; `users` has the columns, `pk_users`, `uq_users_auth0_subject` and `ck_users_auth0_subject_not_blank` from data-model.md; inserting a blank `auth0_subject` fails · skill: backend-developer
- [X] T024 [P] [US1] Write `hl-budget-backend/src/test/kotlin/com/valyalkin/hlbudget/config/VerifiedIdentityResolverTest.kt` (unit, no Spring): blank or missing `sub` → rejected as invalid token; `name` trimmed, blank → null, truncated to 255; `email` trimmed, blank → null, without `@` → null, longer than 320 → null; `picture` not an absolute `https` URL (e.g. `http:`, `javascript:`, relative) → null, longer than 2048 → null; claims read under the configured namespace only · skill: backend-developer
- [X] T025 [P] [US1] Write `hl-budget-backend/src/test/kotlin/com/valyalkin/hlbudget/service/UserServiceTest.kt` (MockK): `ensureProfile` passes the identity to the repository upsert and returns the mapped `User` plus the `created` flag unchanged · skill: backend-developer
- [X] T026 [P] [US1] Write `hl-budget-backend/src/test/kotlin/com/valyalkin/hlbudget/controller/UserControllerIT.kt`: first `PUT /api/v1/users/me` with a valid token → 201, `Location: /api/v1/users/me`, body exactly `id`, `displayName`, `email`, `avatarUrl`, `firstSeenAt`, `lastSeenAt` (no `auth0Subject`, no `version`), `firstSeenAt == lastSeenAt`; second call → 200, same `id`, `lastSeenAt` later, `firstSeenAt` unchanged; a token with a changed name/email updates them on the same row; a token without name/picture claims → `null` fields; timestamps ISO-8601 UTC with `Z` · skill: backend-developer
- [X] T027 [P] [US1] Write `hl-budget-backend/src/test/kotlin/com/valyalkin/hlbudget/controller/UserProvisioningIT.kt` (SC-002, research R-004): fire 10 parallel `PUT /api/v1/users/me` with one new `sub`; all return 2xx, exactly one returns 201, and `select count(*) from users where auth0_subject = ?` is 1 · skill: backend-developer

### Backend implementation for User Story 1

- [X] T028 [US1] Create `hl-budget-backend/src/main/resources/db/migration/V1__create_users.sql`: table `users` with `id uuid not null default gen_random_uuid()`, `auth0_subject varchar(255) not null`, `display_name varchar(255) null`, `email varchar(320) null`, `avatar_url varchar(2048) null`, `first_seen_at timestamptz not null default now()`, `last_seen_at timestamptz not null default now()`, `version bigint not null default 0`; constraints `pk_users` primary key (`id`), `uq_users_auth0_subject` unique (`auth0_subject`), `ck_users_auth0_subject_not_blank` check `length(trim(auth0_subject)) > 0`; no index on `email` · skill: backend-developer
- [X] T029 [P] [US1] Create domain types in `hl-budget-backend/src/main/kotlin/com/valyalkin/hlbudget/model/`: `User.kt` (immutable, annotation-free: `id: UUID`, `subject: String`, `displayName: String?`, `email: String?`, `avatarUrl: String?`, `firstSeenAt: Instant`, `lastSeenAt: Instant`) and `VerifiedIdentity.kt` (`subject: String`, `displayName: String?`, `email: String?`, `avatarUrl: String?`) · skill: backend-developer
- [X] T030 [US1] Create persistence in `hl-budget-backend/src/main/kotlin/com/valyalkin/hlbudget/repository/`: `UserEntity.kt` (JPA entity on `users`, `@Version version`), `UserRepository.kt` with one native statement `INSERT INTO users (auth0_subject, display_name, email, avatar_url) VALUES (…) ON CONFLICT (auth0_subject) DO UPDATE SET display_name = EXCLUDED.display_name, email = EXCLUDED.email, avatar_url = EXCLUDED.avatar_url, last_seen_at = now(), version = users.version + 1 RETURNING *, (xmax = 0) AS inserted` returning an `UpsertResult(user: User, created: Boolean)`, and `UserMappings.kt` (entity/row → `User`) — no find-then-save, no retry, no 409 path (research R-004) · skill: backend-developer
- [X] T031 [US1] Create `hl-budget-backend/src/main/kotlin/com/valyalkin/hlbudget/config/VerifiedIdentityResolver.kt`: builds `VerifiedIdentity` from the validated `Jwt` using `sub` and the claims `<claimsNamespace>name`, `<claimsNamespace>email`, `<claimsNamespace>picture`, applying the data-model.md validation rules exactly (blank `sub` → 401 `unauthenticated`, nothing written; invalid email logged at WARN **without the value**); expose it to the controller as a `HandlerMethodArgumentResolver` for a `VerifiedIdentity` parameter, so no user id can come from the request (FR-015, FR-017) · skill: backend-developer
- [X] T032 [US1] Create `hl-budget-backend/src/main/kotlin/com/valyalkin/hlbudget/service/UserService.kt`: `@Transactional fun ensureProfile(identity: VerifiedIdentity): UpsertResult` calling the repository upsert · skill: backend-developer
- [X] T033 [P] [US1] Create `hl-budget-backend/src/main/kotlin/com/valyalkin/hlbudget/dto/UserProfileResponse.kt` (`id`, `displayName`, `email`, `avatarUrl`, `firstSeenAt`, `lastSeenAt`, camelCase, nullable fields serialised as `null`) and its `User.toResponse()` mapping · skill: backend-developer
- [X] T034 [US1] Create `hl-budget-backend/src/main/kotlin/com/valyalkin/hlbudget/controller/UserController.kt`: `PUT /api/v1/users/me`, no body, no path or query identifier; returns `ResponseEntity` 201 + `Location: /api/v1/users/me` when `created`, else 200; springdoc annotations matching contracts/openapi.yaml: `operationId = "ensureMyProfile"`, tag `Users`, responses 200/201 with the `ProfileReturning`/`ProfileFirstVisit` examples, 401/500/503 problem responses with examples, `X-Request-Id` and `Location` headers · skill: backend-developer
- [X] T035 [US1] Write `hl-budget-backend/src/test/kotlin/com/valyalkin/hlbudget/config/OpenApiExportIT.kt`: fetch `/api/v3/api-docs.yaml`, assert it contains `PUT /api/v1/users/me` with responses 200, 201, 401, 500, 503 and the `UserProfile` schema fields, and write it to `hl-budget-backend/contracts/api.yaml`; commit the generated file. Compare by hand with `specs/001-platform-foundation/contracts/openapi.yaml` and fix any drift in the annotations, not in the file · skill: backend-developer
- [X] T036 [P] [US1] Create the Auth0 Post-Login Action source `hl-budget-backend/auth0/post-login-profile-claims.js`: `exports.onExecutePostLogin` sets access-token custom claims `https://hl-budget/name`, `https://hl-budget/email`, `https://hl-budget/picture` from `event.user.name`, `event.user.email`, `event.user.picture` when present (research R-003), with a header comment on how to deploy it (quickstart §1 step 4) · skill: backend-developer
- [X] T037 [US1] Operator step (ask the user, do not do it yourself): configure the Auth0 **dev/test** tenant per quickstart §1 — API with the audience and RS256; Regular Web Application with callback `http://localhost:3000/auth/callback`, logout URL and web origin `http://localhost:3000`; refresh-token rotation, absolute 30 d, inactivity 7 d; Username-Password connection with **Disable Sign Ups** and Google; the Action from T036 on the Login flow; one seeded test user. Record the non-secret values for `.env.local` · skill: none

### Frontend implementation for User Story 1

- [X] T038 [US1] Copy `hl-budget-backend/contracts/api.yaml` to `hl-budget-frontend/contracts/api.yaml`, run `pnpm generate:api`, and commit the generated `hl-budget-frontend/app/lib/api-schema.d.ts` (never edited by hand) · skill: frontend-developer
- [X] T039 [P] [US1] Create `hl-budget-frontend/app/lib/utils.ts` and `hl-budget-frontend/app/lib/utils.test.ts`: `greeting(profile)` → "Hello, {displayName}" / "Hello, {email}" when `displayName` is null / "Hello" when both are null; `initials(profile)` from `displayName`, else `email`, else `null` (neutral icon) — exactly the data-model.md "UI fallbacks" table · skill: frontend-developer
- [X] T040 [US1] Create `hl-budget-frontend/app/lib/data.ts` (`server-only`): `ensureMyProfile()` calls `authedFetch('/api/v1/users/me', { method: 'PUT' })` and returns the `UserProfile` type from `api-schema.d.ts`, treating 200 and 201 the same; `hl-budget-frontend/app/lib/data.test.ts` (MSW: 200 and 201 both return the profile; `PUT` method and bearer header used) · skill: frontend-developer
- [X] T041 [P] [US1] Create `hl-budget-frontend/app/ui/home/avatar.tsx` (`'use client'` leaf: renders `<img>` from `avatarUrl` with `alt` and `referrerPolicy="no-referrer"`; on `onError` or null URL shows the initials placeholder, or a neutral icon when there are no initials) and `hl-budget-frontend/app/ui/home/avatar.test.tsx` · skill: frontend-developer
- [X] T042 [P] [US1] Create `hl-budget-frontend/app/ui/home/profile-card.tsx` (server component: greeting, email when present, `Avatar`) with `hl-budget-frontend/app/ui/home/profile-card.test.tsx` covering all four fallback rows, and `hl-budget-frontend/app/ui/skeletons.tsx` (`ProfileCardSkeleton`) · skill: frontend-developer
- [X] T043 [US1] Create `hl-budget-frontend/app/home/page.tsx` (metadata `title: 'Home'`, `robots: { index: false, follow: false }`; header with "HL Budget"; an async `Profile` server component calling `ensureMyProfile()` inside `<Suspense fallback={<ProfileCardSkeleton/>}>`) and `hl-budget-frontend/app/home/loading.tsx` · skill: frontend-developer
- [X] T044 [US1] Create the landing page `hl-budget-frontend/app/page.tsx` (metadata title "HL Budget", indexable; heading "HL Budget", one-line description, no user data) and `hl-budget-frontend/app/ui/landing/landing-actions.tsx` (without a session: **Sign in** as `<a href="/auth/login">`, no prefetch; with a session: **Go to HL Budget** linking to `/home`), with `hl-budget-frontend/app/ui/landing/landing-actions.test.tsx` · skill: frontend-developer
- [X] T045 [US1] Write `hl-budget-frontend/e2e/sign-in.spec.ts` against the local stack and the dev tenant test user (credentials from env, never committed): landing shows name, description and Sign in; Sign in → Universal Login → `/home` shows greeting, email and avatar (SC-001 loose timing); two browser contexts signing in at once both reach `/home` (SC-002) · skill: frontend-developer

**Checkpoint**: Quickstart L1–L5 pass locally; backend and frontend gates green. MVP done.

---

## Phase 4: User Story 2 — Protected pages require sign-in (Priority: P1)

**Goal**: Signed-out visitors are sent to sign in and returned to the page they asked for (same
origin only); the API refuses every request without valid credentials and creates nothing.

**Independent Test**: Signed out, open `/home?x=1` → login → back on `/home?x=1`. Call the API with
no token and with each kind of invalid token → 401, no row created.

### Tests for User Story 2 ⚠️

- [X] T046 [P] [US2] Write `hl-budget-backend/src/test/kotlin/com/valyalkin/hlbudget/config/SecurityIT.kt` (SC-003, research R-005): `PUT /api/v1/users/me` with no token, malformed, expired, wrong issuer, wrong audience, missing `aud`, unknown signing key and blank `sub` → each 401 `application/problem+json`, `type` `urn:problem:hl-budget:unauthenticated`, `WWW-Authenticate` present, `traceId` == `X-Request-Id`, and the `users` table stays empty; a valid token → 201; `/healthz`, `/readyz` and `/api/v3/api-docs` reachable without a token · skill: backend-developer
- [X] T047 [P] [US2] Write `hl-budget-frontend/app/lib/safe-return-to.test.ts`: accepts `/home`, `/home?x=1`; rejects `https://example.com`, `//evil.example`, `/\evil.example`, `\\evil.example`, `javascript:alert(1)`, empty and missing → returns `/home` · skill: frontend-developer
- [X] T048 [P] [US2] Write `hl-budget-frontend/proxy.test.ts` (mock `auth0`): no session on `/home?x=1` → 302 to `/auth/login?returnTo=%2Fhome%3Fx%3D1` with an empty body; with a session → pass-through with `Cache-Control: private, no-cache, no-store`; `/` passes without a session; matcher excludes `healthz` and `readyz` · skill: frontend-developer

### Implementation for User Story 2

- [X] T049 [US2] Create `hl-budget-frontend/app/lib/safe-return-to.ts`: `safeReturnTo(value, appBaseUrl)` returns the value only if it is a relative path starting with a single `/`, contains no backslash, and resolves to the same origin as `APP_BASE_URL`; otherwise `/home` (FR-003, US2-AS4) · skill: frontend-developer
- [X] T050 [US2] Add the `onCallback` hook to `hl-budget-frontend/app/lib/auth0.ts`: on success redirect to `safeReturnTo(context.returnTo)` (research R-009). Leave a clear spot for the error branch added in T060 · skill: frontend-developer
- [X] T051 [P] [US2] Create `hl-budget-frontend/app/robots.ts` (allow `/`, disallow `/home` and `/auth/`) and `hl-budget-frontend/app/sitemap.ts` (only `/`, from `APP_BASE_URL`), with a test asserting `/home` is absent from the sitemap (FR-004) · skill: frontend-developer
- [X] T052 [US2] Write `hl-budget-frontend/e2e/protected-routes.spec.ts` (SC-003): signed out, `/home?x=1` → login page with no profile content, then after login on `/home?x=1`; a tampered `__session` cookie → sent to login; `/auth/login?returnTo=https://example.com` → after login on `/home`; `/home` response has `X-Robots-Tag` or meta `noindex` · skill: frontend-developer

**Checkpoint**: US1 and US2 both pass on their own; `SecurityIT` green.

---

## Phase 5: User Story 3 — Sign out (Priority: P2)

**Goal**: "Sign out" ends the session in HL Budget and Auth0 and returns to the landing page;
afterwards protected pages need a fresh sign-in.

**Independent Test**: Sign in, sign out, land on `/`; Back shows no profile; `/home` asks to sign in;
Auth0 asks for credentials again.

- [X] T053 [P] [US3] Create `hl-budget-frontend/app/ui/home/sign-out-link.tsx` (`<a href="/auth/logout">Sign out</a>`, a plain link, not a client fetch) and `hl-budget-frontend/app/ui/home/sign-out-link.test.tsx` · skill: frontend-developer
- [X] T054 [US3] Add `SignOutLink` to the header in `hl-budget-frontend/app/home/page.tsx` (FR-009: present on every protected page) · skill: frontend-developer
- [X] T055 [US3] In `hl-budget-frontend/app/lib/auth0.ts`, make sure logout ends the Auth0 session through OIDC RP-initiated logout and returns to `APP_BASE_URL` (`logoutStrategy` per the SDK 4.32 docs and the skill's `references/auth.md`); confirm the logout URL is in the dev tenant's Allowed Logout URLs (T037) · skill: frontend-developer
- [X] T056 [US3] Write `hl-budget-frontend/e2e/sign-out.spec.ts`: sign in → Sign out → on `/` showing **Sign in**; browser Back shows no profile content (page reloads and redirects); `/home` → login page; starting sign-in shows the credentials form rather than signing in silently; the `/home` response carried `Cache-Control: private, no-cache, no-store` (US3-AS4) · skill: frontend-developer

**Checkpoint**: US3 passes on its own; US1/US2 specs still green.

---

## Phase 6: User Story 4 — Failed sign-in, expired sessions and errors handled gracefully (Priority: P2)

**Goal**: Cancelled sign-in, expired session, API 401, API down/slow and unknown URLs each end in a
friendly message or a sign-in redirect — never a raw error.

**Independent Test**: Cancel login → landing notice; reject credentials → sent to sign in; stop the
backend → "Something went wrong" + Try again, which recovers; `/nope` → not-found page.

### Tests for User Story 4 ⚠️

- [X] T057 [P] [US4] Extend `hl-budget-frontend/app/lib/data.test.ts`: API 401 → `redirect('/auth/login?returnTo=/home')` is called (mock `next/navigation`); 500 problem, timeout and connection refused → `ensureMyProfile()` throws · skill: frontend-developer
- [X] T058 [P] [US4] Write `hl-budget-backend/src/test/kotlin/com/valyalkin/hlbudget/error/ServiceUnavailableIT.kt`: with the Testcontainers Postgres stopped (or a repository mock throwing `CannotGetJdbcConnectionException`/`DataAccessResourceFailureException`), `PUT /api/v1/users/me` with a valid token → 503 `urn:problem:hl-budget:service-unavailable`, detail "The service is temporarily unavailable", no internals · skill: backend-developer

### Implementation for User Story 4

- [X] T059 [US4] Map database-connection failures (`CannotGetJdbcConnectionException`, `DataAccessResourceFailureException`, `CannotCreateTransactionException`) to 503 `service-unavailable` with the fixed detail in `hl-budget-backend/src/main/kotlin/com/valyalkin/hlbudget/error/GlobalExceptionHandler.kt`, logging the cause against `traceId` · skill: backend-developer
- [X] T060 [US4] Add the error branch of `onCallback` in `hl-budget-frontend/app/lib/auth0.ts`: any callback error (`access_denied`, cancelled, failed) redirects to `/?signin=failed` (FR-006); add a unit test for both branches in `hl-budget-frontend/app/lib/auth0.test.ts` · skill: frontend-developer
- [X] T061 [P] [US4] Create `hl-budget-frontend/app/ui/landing/sign-in-notice.tsx` (`role="status"` / `aria-live="polite"` text "Sign-in did not complete. You can try again.") with a test, and render it from `hl-budget-frontend/app/page.tsx` when `searchParams.signin === 'failed'` · skill: frontend-developer
- [X] T062 [US4] Update `hl-budget-frontend/app/lib/data.ts`: catch `UnauthenticatedError` and call `redirect('/auth/login?returnTo=/home')` (FR-010, US4-AS3); let `ApiError` and `ApiUnavailableError` propagate to the error boundary · skill: frontend-developer
- [X] T063 [P] [US4] Create `hl-budget-frontend/app/home/error.tsx` (`'use client'`; heading "Something went wrong", a **Try again** button calling `reset()` then `router.refresh()`, an optional short "Reference: <digest/traceId>" line, no status code or stack) with `hl-budget-frontend/app/home/error.test.tsx` (FR-021, US4-AS4/5) · skill: frontend-developer
- [X] T064 [P] [US4] Create `hl-budget-frontend/app/global-error.tsx` (own `<html>`/`<body>`, same friendly message and Try again, no technical details) · skill: frontend-developer
- [X] T065 [P] [US4] Create `hl-budget-frontend/app/not-found.tsx` (metadata `title: 'Not found'`, `robots: { index: false }`; message and a link back to `/`) with `hl-budget-frontend/app/not-found.test.tsx` (FR-022) · skill: frontend-developer
- [X] T066 [US4] Write `hl-budget-frontend/e2e/failures.spec.ts` (SC-006): `/auth/callback?error=access_denied&state=…` path → landing with the notice; signed in with the backend stopped (or `API_BASE_URL` pointed at a closed port) → "Something went wrong" with Try again and no stack/status text; backend restored → Try again shows the profile; an expired session cookie → sent to login and back to `/home` · skill: frontend-developer
- [X] T067 [P] [US4] Write `hl-budget-frontend/e2e/not-found.spec.ts`: `/nope` signed out and signed in → not-found page with a working link to `/`, HTTP 404 · skill: frontend-developer

**Checkpoint**: All failure paths covered; US1–US3 still green.

---

## Phase 7: User Story 5 — Operator sees both services healthy after a deploy (Priority: P2)

**Goal**: Both services expose `/healthz` and `/readyz`, run in hardened containers, ship through
release workflow → ghcr → hl-helm PR → Argo CD, and load over trusted HTTPS on one private host.

**Independent Test**: After merging a release, Argo CD shows `hl-budget` Synced/Healthy, `https://<host>/`
loads with a valid certificate; stopping Postgres (locally) flips the API's `/readyz` to 503 and
back.

### Tests for User Story 5 ⚠️

- [X] T068 [P] [US5] Write `hl-budget-backend/src/test/kotlin/com/valyalkin/hlbudget/config/HealthIT.kt`: `GET /healthz` and `GET /readyz` → 200 `{"status":"UP"}` without a token and with no `components`/details in the body · skill: backend-developer
- [X] T069 [P] [US5] Write `hl-budget-backend/src/test/kotlin/com/valyalkin/hlbudget/config/ReadinessIT.kt` (SC-007): with its own Postgres container, stop it → `/readyz` returns 503 within 30 s while `/healthz` stays 200; start it → `/readyz` 200 within 30 s (use Awaitility or a polling loop) · skill: backend-developer
- [X] T070 [P] [US5] Write `hl-budget-frontend/app/healthz/route.test.ts` and `hl-budget-frontend/app/readyz/route.test.ts`: both return 200 `{"status":"UP"}` · skill: frontend-developer

### Implementation for User Story 5

- [X] T071 [US5] Configure health in `hl-budget-backend/src/main/resources/application.yaml` (research R-008): `management.endpoints.web.exposure.include: health`, `management.endpoint.health.probes.enabled: true`, `show-details: never`, `show-components: never`, `management.endpoint.health.group.liveness.additional-path: server:/healthz`, `management.endpoint.health.group.readiness.additional-path: server:/readyz`, readiness group `include: readinessState,db`; liveness group checks no dependency · skill: backend-developer
- [X] T072 [P] [US5] Create `hl-budget-frontend/app/healthz/route.ts` and `hl-budget-frontend/app/readyz/route.ts`: `GET` → `Response.json({ status: 'UP' })`, `dynamic = 'force-dynamic'`, no backend call (research R-008) · skill: frontend-developer
- [X] T073 [US5] Containerize the backend: run the `hl-containerize` skill in `hl-budget-backend/` to write `hl-budget-backend/Dockerfile` and `hl-budget-backend/.dockerignore` per research R-015 (build on pinned `eclipse-temurin:21-jdk` with `./gradlew bootJar -x check`; runtime pinned `eclipse-temurin:21-jre` by version + digest, `USER 10001:10001`, `EXPOSE 8080`, `ENV JAVA_TOOL_OPTIONS=-Djava.io.tmpdir=/tmp`, `ENTRYPOINT ["java","-XX:MaxRAMPercentage=75","-jar","/app/app.jar"]`; no `.env`, no secrets). Proof: `--read-only --tmpfs /tmp --user 10001:10001 --cap-drop ALL --security-opt no-new-privileges`; `/healthz` 200, `/readyz` 503 without a database, `docker stop` < 10 s; tmpDirs `[/tmp]` · skill: homelab:hl-containerize
- [X] T074 [P] [US5] Containerize the frontend: run the `hl-containerize` skill in `hl-budget-frontend/` to write `hl-budget-frontend/Dockerfile` and `hl-budget-frontend/.dockerignore` from `templates/Dockerfile.node` adapted to pnpm + `output: 'standalone'` (runtime pinned `node:22-alpine`, `USER 10001:10001`, `ENV PORT=8080 HOSTNAME=0.0.0.0`, `CMD ["node","server.js"]`, no `pnpm start` wrapper). Proof as in T073; `/healthz` and `/readyz` 200; tmpDirs `[/tmp, /app/.next/cache]` · skill: homelab:hl-containerize
- [X] T075 [US5] Gate and merge: confirm `./gradlew check` (backend) and `pnpm typecheck && pnpm lint && pnpm test && pnpm build` (frontend) are green locally and in CI, open a PR from `001-platform-foundation` to `main` in each service repo, and **ask the user before merging** (a merge to `main` releases) · skill: none
- [X] T076 [US5] Onboard the backend: run `hl-onboard-service` from `hl-budget-backend/` with app `hl-budget`, service `backend`, image `hl-budget-backend` (research R-001; hl-helm found at `~/Documents/Projects/hl-helm`). It writes `hl-budget-backend/homelab.yaml` (platforms from `kubectl get nodes`), copies `hl-budget-backend/.github/workflows/release.yaml` unchanged, pushes the first image, and opens the hl-helm PR creating `~/Documents/Projects/hl-helm/charts/apps/hl-budget/` (`Chart.yaml`, `values.yaml`, `values.schema.json`, `README.md`, `ci/default-values.yaml`, `templates/namespace.yaml`, `templates/backend.yaml`) with the `backend` block from contracts/deployment.md (env `SPRING_DATASOURCE_URL`, `HLBUDGET_AUTH_*`, `LOGGING_LEVEL_ROOT`; 512Mi; tmpDirs `[/tmp]`; probes liveness 10 s, readiness 5 s, startup `failureThreshold: 60, periodSeconds: 2`; ingress host `<host>` path `/api`; Postgres credentials keys `{username: SPRING_DATASOURCE_USERNAME, password: SPRING_DATASOURCE_PASSWORD}`) and `- name: hl-budget` in `~/Documents/Projects/hl-helm/bootstrap/values/postgres.yaml` · skill: homelab:hl-onboard-service
- [X] T077 [US5] Onboard the frontend: run `hl-onboard-service` from `hl-budget-frontend/` to extend the same app with the `frontend` block in `~/Documents/Projects/hl-helm/charts/apps/hl-budget/values.yaml` and `templates/frontend.yaml` per contracts/deployment.md (env `APP_BASE_URL`, `API_BASE_URL=http://hl-budget-backend`, `AUTH0_DOMAIN`, `AUTH0_CLIENT_ID`, `AUTH0_AUDIENCE`, `API_TIMEOUT_MS`; 256Mi; tmpDirs `[/tmp, /app/.next/cache]`; ingress same `<host>` path `/`; secrets `auth0` from `apps/hl-budget/frontend` and `tls` from `apps/hl-budget/tls` of type `kubernetes.io/tls`, `keys: {}`, `rolloutRestart: false`); set `ingress.tls.secretName: hl-budget-frontend-tls` on **both** ingresses (research R-012), or do the TLS part as a follow-up with `hl-service-config`. Writes `hl-budget-frontend/homelab.yaml` and `hl-budget-frontend/.github/workflows/release.yaml` (unchanged template). Note the cert-renewal follow-up in the PR body · skill: homelab:hl-onboard-service
- [X] T078 [US5] Operator checklist (ask the user to do these; never write secrets yourself): configure the Auth0 **prod** tenant per quickstart §1 with `https://<host>` URLs; make both ghcr packages public; fill Vault `kv/apps/hl-budget/frontend` (`auth0_client_secret`, `auth0_secret`) and `kv/apps/hl-budget/tls` (`tls.crt`, `tls.key`); merge the hl-helm PR(s); run `bootstrap/install.sh --only postgres --diff` then apply; add the internal DNS record for `<host>` on the home and VPN resolvers; set `HL_HELM_TOKEN` in both service repos · skill: none · Closed 2026-10-26: done except the deferred items (prod Auth0 tenant, TLS certificate in `kv/apps/hl-budget/tls`, DNS on the home and VPN resolvers), to be handled later
- [X] T079 [US5] Verify the deploy with `hl-deploy-status` for `hl-budget/backend` and `hl-budget/frontend` (workflow → PR → Argo CD → pods → Vault sync), then walk quickstart D1–D7 and record results; fix chart issues through `hl-service-config` · skill: homelab:hl-deploy-status

**Checkpoint**: `hl-budget` Synced/Healthy; app loads on `https://<host>/` with a trusted certificate (SC-004).

---

## Phase 8: User Story 6 — Developer runs everything locally (Priority: P3)

**Goal**: A developer can run Postgres, the API and the web app locally from documented steps,
with every setting documented and no real secrets committed.

**Independent Test**: On a clean checkout, follow the READMEs with a dev tenant and sign in locally
within 30 minutes (SC-008); a missing setting fails startup naming it.

- [X] T080 [P] [US6] Write `hl-budget-backend/src/test/kotlin/com/valyalkin/hlbudget/config/HlBudgetPropertiesTest.kt` (`ApplicationContextRunner`): each missing `hlbudget.auth.*` property fails context start with a message naming that property (US6-AS3) · skill: backend-developer
- [X] T081 [P] [US6] Create `hl-budget-backend/.env.example` listing exactly the variables in contracts/deployment.md "hl-budget-backend" (`PORT`, `SPRING_DATASOURCE_URL`, `SPRING_DATASOURCE_USERNAME`, `SPRING_DATASOURCE_PASSWORD`, `HLBUDGET_AUTH_ISSUER`, `HLBUDGET_AUTH_AUDIENCE`, `HLBUDGET_AUTH_JWKSETURI`, `HLBUDGET_AUTH_CLAIMSNAMESPACE`), each with a one-line comment (purpose, secret or not, required or not) and the placeholder values from that table only · skill: backend-developer
- [X] T082 [P] [US6] Create `hl-budget-frontend/.env.example` listing exactly the variables in contracts/deployment.md "hl-budget-frontend" (`PORT`, `APP_BASE_URL`, `API_BASE_URL`, `API_TIMEOUT_MS`, `AUTH0_DOMAIN`, `AUTH0_CLIENT_ID`, `AUTH0_CLIENT_SECRET`, `AUTH0_SECRET`, `AUTH0_AUDIENCE`), each commented, placeholder values only, no `NEXT_PUBLIC_*` · skill: frontend-developer
- [X] T083 [P] [US6] Write `hl-budget-backend/README.md`: prerequisites, Auth0 dev setup pointer (incl. the Action in `auth0/`), `cp .env.example .env.local`, `./gradlew bootRun --args='--spring.profiles.active=local'` (compose starts Postgres 17.7), gate command, health and OpenAPI URLs, `contracts/api.yaml` export via `OpenApiExportIT` · skill: backend-developer
- [X] T084 [P] [US6] Write `hl-budget-frontend/README.md`: prerequisites, `cp .env.example .env.local`, `AUTH0_SECRET=$(openssl rand -hex 32)`, `pnpm install && pnpm dev`, regenerating types (`copy contracts/api.yaml` + `pnpm generate:api`), gate commands, running Playwright against the dev tenant · skill: frontend-developer
- [X] T085 [US6] Run quickstart §2 checks L1–L7 on a clean checkout of both worktrees and record the results; fix any README gaps found · skill: none

**Checkpoint**: Local setup reproducible; no real secrets in either repo.

---

## Phase 9: Polish & Cross-Cutting Concerns

**Purpose**: Final gate runs, contract freshness and cleanup across both repos.

- [X] T086 [P] Check the generated contract matches the design: diff `hl-budget-backend/contracts/api.yaml` against `specs/001-platform-foundation/contracts/openapi.yaml` (paths, status codes, schemas, examples, problem URNs); re-copy to `hl-budget-frontend/contracts/api.yaml` and confirm `pnpm generate:api && git diff --exit-code app/lib/api-schema.d.ts` is clean · skill: frontend-developer
- [X] T087 [P] Scope check (FR-028): grep both repos for budgeting, settings, roles or admin code and remove anything found; confirm the only business endpoint is `PUT /api/v1/users/me` · skill: none
- [X] T088 Run all gates: `cd hl-budget-backend && ./gradlew check` (Kover 80/75, 75 per package); `cd hl-budget-frontend && pnpm typecheck && pnpm lint && pnpm test && pnpm build && pnpm e2e`; gitleaks in both CI runs green · skill: none
- [X] T089 Run `quickstart.md` §3 and §6 journey checks on the deployed app and note any failure as a new task before calling the feature done · skill: none

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: none — start immediately.
- **Foundational (Phase 2)**: depends on Setup — **blocks all user stories**.
- **US1 (Phase 3)**: depends on Foundational. The backend half (T023–T036) lands before the
  frontend half (T038–T045), because the frontend generates its types from the backend contract
  (Development Workflow: "contract lands first"). T037 (Auth0 dev tenant) blocks the e2e spec T045.
- **US2 (Phase 4)**: depends on Foundational. `SecurityIT` (T046) needs only T013/T015 plus the
  controller (T034) for the 201 case; the frontend part needs T018/T019. Its e2e (T052) needs US1's
  `/home`.
- **US3 (Phase 5)**: depends on US1 (`/home` page header).
- **US4 (Phase 6)**: depends on US1 (`data.ts`, `/home`) and on US2's `onCallback` (T050) for T060.
- **US5 (Phase 7)**: health tasks (T068–T072) depend only on Foundational; containers (T073/T074)
  after health; T075 merge after all P1/P2 stories are green; onboarding T076 → T077 → T078 → T079.
- **US6 (Phase 8)**: depends on Foundational; T085 needs US1 done.
- **Polish (Phase 9)**: after all desired stories.

### User Story Dependencies

```text
Setup ─▶ Foundational ─┬─▶ US1 (P1, MVP) ─┬─▶ US3 (P2)
                       │                  ├─▶ US4 (P2)  (also needs US2 T050)
                       ├─▶ US2 (P1) ──────┘
                       ├─▶ US5 health/containers ─▶ merge (T075) ─▶ onboarding ─▶ verify
                       └─▶ US6 (P3)
```

### Within Each User Story

- Tests are written first and fail before the code that makes them pass.
- Migration → domain → repository → service → controller → contract export → frontend types →
  data function → UI → e2e.
- `auth0.ts` is edited in T018, T050, T055, T060 and `app/page.tsx` in T044, T061 — do these in
  order, never in parallel.

### Parallel Opportunities

- Setup: T003, T004, T005 in parallel after T002; T007, T008, T009 in parallel after T006. The
  backend (T002–T005) and frontend (T006–T009) tracks run in parallel.
- Foundational: T011, T012, T014 in parallel; T017, T020 (after T018 for `getAccessToken`), T021,
  T022 in parallel. Backend and frontend tracks run in parallel.
- US1: tests T023–T027 together; T029, T033, T036 in parallel; T039, T041, T042 in parallel.
- US2: T046, T047, T048 together; T051 alongside T049.
- US4: T057, T058 together; T061, T063, T064, T065, T067 in parallel.
- US5: T068, T069, T070 together; T073 and T074 in parallel.
- US6: T080–T084 all in parallel.

---

## Parallel Example: User Story 1

```bash
# Backend tests for US1, all at once (different files):
Task: "SchemaValidationIT in hl-budget-backend/src/test/kotlin/com/valyalkin/hlbudget/repository/SchemaValidationIT.kt"
Task: "VerifiedIdentityResolverTest in hl-budget-backend/src/test/kotlin/com/valyalkin/hlbudget/config/VerifiedIdentityResolverTest.kt"
Task: "UserServiceTest in hl-budget-backend/src/test/kotlin/com/valyalkin/hlbudget/service/UserServiceTest.kt"
Task: "UserControllerIT in hl-budget-backend/src/test/kotlin/com/valyalkin/hlbudget/controller/UserControllerIT.kt"
Task: "UserProvisioningIT in hl-budget-backend/src/test/kotlin/com/valyalkin/hlbudget/controller/UserProvisioningIT.kt"

# Frontend leaves for US1, after the types are generated (T038):
Task: "utils.ts + utils.test.ts in hl-budget-frontend/app/lib/"
Task: "avatar.tsx + avatar.test.tsx in hl-budget-frontend/app/ui/home/"
Task: "profile-card.tsx + skeletons.tsx in hl-budget-frontend/app/ui/"
```

## Parallel Example: User Story 4

```bash
Task: "sign-in-notice.tsx in hl-budget-frontend/app/ui/landing/"
Task: "error.tsx in hl-budget-frontend/app/home/"
Task: "global-error.tsx in hl-budget-frontend/app/"
Task: "not-found.tsx in hl-budget-frontend/app/"
Task: "not-found.spec.ts in hl-budget-frontend/e2e/"
```

---

## Implementation Strategy

### MVP First (User Story 1)

1. Phase 1 Setup → Phase 2 Foundational.
2. Phase 3 US1: backend contract first, then frontend.
3. **Stop and validate**: quickstart L1–L5 locally, both gates green.

### Incremental Delivery

1. Setup + Foundational → foundation ready.
2. US1 → MVP (sign in, see profile).
3. US2 → protection proven (with US1, the P1 set is complete).
4. US3 + US4 → sign-out and failure paths.
5. US5 → containers, merge, onboard, deploy, verify (the feature's "deployed and healthy" done).
6. US6 → local docs and fail-fast proof (can run alongside US3–US5).
7. Polish.

Constitution "done" for this feature needs P1 + P2 stories, all gates green, hardened containers
proven, and Argo CD reporting healthy (US5). US6 completes the local-developer path.

---

## Notes

- [P] tasks touch different files and depend on nothing unfinished.
- The `[Story]` label maps each task to a spec user story for traceability.
- Tasks marked `skill: none` that say "ask the user" are operator steps: stop, ask, and wait.
- Never write real secrets anywhere; Auth0 test-user credentials for Playwright come from local
  env and CI secrets only.
- Merging to `main` in a service repo releases it. Ask first.
