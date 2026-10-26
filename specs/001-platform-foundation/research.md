# Research: Platform Foundation

**Feature**: `001-platform-foundation` | **Date**: 2026-10-25 | **Plan**: [plan.md](./plan.md)

Each decision below resolves an unknown from the plan's Technical Context, or settles a point
where the constitution, the `backend-developer` / `frontend-developer` skills and the `homelab:*`
skills disagree. Where they conflicted, the constitution won, and the reason is recorded.

---

## R-001: Repositories, application and service names

- **Decision**: Two existing (empty, created 2026-10-25) GitHub repositories:
  `valyalkin/hl-budget-backend` and `valyalkin/hl-budget-frontend`. Each has a main clone in
  `~/Documents/Projects/<repo>` (branch `main`) and a **git worktree** on the spec branch
  `001-platform-foundation` at `hl-budget-spec/<repo>/`, so one session sees the spec and both
  codebases. The worktrees are listed in this repo's `.gitignore` (git does not track them) and
  re-included in `.ignore` (so code search still finds them). This repository keeps the
  constitution and specs; code is committed only in the service repos. Homelab application
  `hl-budget` (one chart `charts/apps/hl-budget`, namespace `hl-budget`). One name per service
  everywhere: repo = image = in-cluster name, and service block = the repo's suffix.

  | Repo | Service block | Image | In-cluster name |
  |---|---|---|---|
  | `hl-budget-backend` | `backend` | `ghcr.io/valyalkin/hl-budget-backend` | `hl-budget-backend` |
  | `hl-budget-frontend` | `frontend` | `ghcr.io/valyalkin/hl-budget-frontend` | `hl-budget-frontend` |

  This is the homelab's default image name (`<app>-<service>`), so nothing overrides it.
- **Rationale**: The constitution (Principle II) and `homelab.md` both require one service per
  repository and one image per service. A worktree per spec branch keeps each repo's `main`
  deployable (Development Workflow) while `/speckit-implement` edits code from this repo. From a
  worktree, `../hl-helm` does not exist, so `hl-onboard-service` finds hl-helm at its third
  location, `~/Documents/Projects/hl-helm`. One application keeps both services in one
  namespace, sharing one hostname inside one chart (hl-helm forbids a host shared *across*
  charts). Release workflows trigger on `main`, so nothing deploys until a feature branch is
  merged in the service repo.
- **Alternatives considered**: A monorepo with `frontend/` and `backend/` — rejected by
  Principle II and by `homelab.yaml` (one per repo). Two apps (`hl-budget-backend`, `hl-budget-frontend`)
  — rejected: two charts cannot share a host.
- **Confirm at onboarding**: `hl-onboard-service` phase 2 asks for app/service/image names. Use
  the values above unless the operator changes them then; the Postgres database and role will be
  named after the app (`hl-budget`).

## R-002: Profile operation — method, path and status codes

- **Decision**: `PUT /api/v1/users/me`, no request body. Returns `201 Created` with
  `Location: /api/v1/users/me` when this call created the record, `200 OK` otherwise. Body is the
  `UserProfile` (see [contracts/openapi.yaml](./contracts/openapi.yaml)).
- **Rationale**: FR-012/FR-013 require the call to create the record on first use and refresh
  `lastSeenAt`, name, email and avatar every time. Principle VII says `GET` is safe and never
  changes state, so a `GET` that inserts rows violates the constitution. `PUT` is the idempotent
  "make my record match my verified identity" operation: repeating it yields the same record
  (only `lastSeenAt` moves), which is exactly the semantics of FR-014. Principle VII requires
  `201` + `Location` on create, so the first call returns 201 (the backend skill's one sanctioned
  `ResponseEntity`). `/users/me` keeps a plural noun resource with a self alias and no verbs.
  The operation takes no identifier (FR-017): `me` is resolved from the token's `sub` only.
- **Alternatives considered**:
  - `GET /api/v1/users/me` with lazy provisioning — simplest, but breaks "GET never changes
    state" (Principle VII). Rejected.
  - `POST /api/v1/users` — returns 409 on the second call, so the frontend needs two calls and
    a race path. Rejected.
  - `GET` + a separate `PUT` on sign-in only — two operations where the spec asks for one, and
    the record would not refresh on later page loads. Rejected for this feature; a read-only
    `GET /api/v1/users/me` can be added later without breaking the contract.
- **Contract compatibility**: new API, `v1`. Nothing to be backward compatible with.

## R-003: Where the backend gets name, email and avatar

- **Decision**: An Auth0 **Post-Login Action** adds namespaced custom claims to the access token
  for the HL Budget API audience: `https://hl-budget/name`, `https://hl-budget/email`,
  `https://hl-budget/picture`. The backend reads them from the validated JWT only. The namespace
  is configuration (`hlbudget.auth.claims-namespace`, env `HLBUDGET_AUTH_CLAIMSNAMESPACE`,
  value `https://hl-budget/`).
- **Rationale**: Principle III and FR-015 say the backend identifies the user only from verified
  credentials and trusts nothing else from the caller. An access token issued for an API
  audience carries no profile claims by default (`frontend-developer` `references/auth.md`).
  Claims inside the signed token are verified by the same signature check as `sub`, and need no
  extra network call per request. Auth0 drops non-namespaced claims that collide with reserved
  ones, hence the namespace.
- **Alternatives considered**: Backend calls Auth0 `/userinfo` with the bearer token — adds an
  outbound dependency, latency and a failure mode to every profile call, and Auth0 rate-limits
  it. Rejected. Frontend sends the profile in the request body — violates FR-015. Rejected.
- **Missing claims**: name and picture may be absent (edge case "missing profile fields"); email
  may also be absent if the Action is misconfigured. All three are stored as nullable, and the UI
  falls back (see data-model.md). The Action is versioned in `hl-budget-backend/auth0/` as source
  and applied by the operator (quickstart step 1).

## R-004: Exactly one user record under concurrent first calls (FR-014, SC-002)

- **Decision**: A unique constraint on `users.auth0_subject` plus a single native
  `INSERT … ON CONFLICT (auth0_subject) DO UPDATE SET … RETURNING *, (xmax = 0) AS inserted`
  statement in the repository. `inserted` tells the service whether to answer 201 or 200.
- **Rationale**: One atomic statement cannot lose a race: two first-time requests both succeed,
  one inserts, one updates. The backend skill's "lost-race 409" pattern is for creates that must
  refuse a duplicate; here both callers must get their profile, so a 409 would be a defect.
  `xmax = 0` is the standard Postgres signal for "this row was inserted by this statement".
- **Alternatives considered**: find-then-save with retry on `DataIntegrityViolationException`
  — two round trips, a retry loop and a 409 path to test. Rejected. Serializable isolation —
  heavier and still needs a retry. Rejected.
- **Test**: an integration test fires N parallel `PUT /api/v1/users/me` with one token against
  Testcontainers Postgres and asserts one row and N 2xx responses (one 201).

## R-005: Backend token validation (FR-015, FR-016)

- **Decision**: Spring Security OAuth2 resource server with `NimbusJwtDecoder.withJwkSetUri`, a
  `DelegatingOAuth2TokenValidator` of issuer + timestamps (default) **and an explicit audience
  validator that accepts a null `aud`**, both 401 entry points writing problem+json in UTF-8,
  and the request-id filter at order `-110`. Exactly the `backend-developer` `references/auth.md`
  pattern. No scopes are required in this feature: any valid token for the audience may call
  `PUT /api/v1/users/me` (FR-019, no allow-list).
- **Rationale**: Spring's default validator does not check `aud`; omitting it would accept
  tokens for other APIs on the same tenant. `withJwkSetUri` lets the service start while Auth0
  is unreachable (readiness depends only on Postgres, see R-008).
- **Tests**: `SecurityIT` with real RSA-signed tokens and a WireMock-served JWKS: no token,
  malformed, expired, wrong issuer, wrong audience, missing `aud`, bad signature → 401
  problem+json with `traceId`; valid → 200/201; no user row created on any 401 (SC-003).

## R-006: Backend path layout behind a shared host

- **Decision**: Business endpoints are mapped at `/api/v1/...` in the controllers (no servlet
  context path). The OpenAPI document is served at `/api/v3/api-docs` (+ `.yaml`) and Swagger UI
  at `/api/swagger-ui.html`, so they are reachable through the `/api` ingress. Health is served
  at `/healthz` and `/readyz` on the main port (R-008); those paths are reached only by the
  kubelet, not through the ingress.
  On the cluster the chart sets `SPRINGDOC_API_DOCS_ENABLED=false` and
  `SPRINGDOC_SWAGGER_UI_ENABLED=false`, so neither is public there (a 404 problem, pinned by
  `ApiDocsDisabledIT`); they stay on locally and in tests, where `OpenApiExportIT` exports the
  contract.
- **Rationale**: Traefik does not strip `/api` (`service-contract.md`), and Principle VII puts
  the version right after `/api`. A context path of `/api` would also move `/healthz` to
  `/api/healthz`, which breaks the homelab probe convention.
- **Alternatives considered**: `server.servlet.context-path=/api` — moves health endpoints.
  Rejected. A Traefik strip-prefix middleware — not offered by `hl-common` and would need a
  sanctioned-mechanism extension. Rejected.

## R-007: Backend database wiring under the homelab chart

- **Decision**: No `spring.datasource` block in `application.yaml` (backend skill rule). In the
  chart, the Postgres dependency's credential keys map straight onto Spring's names —
  `keys: {username: SPRING_DATASOURCE_USERNAME, password: SPRING_DATASOURCE_PASSWORD}` — and
  `workload.env` sets
  `SPRING_DATASOURCE_URL=jdbc:postgresql://postgres-rw.postgres.svc.cluster.local:5432/hl-budget`.
  Locally, `spring-boot-docker-compose` provides the connection; in tests, `@ServiceConnection`.
- **Rationale**: Reconciles the backend skill ("no datasource block; production comes from
  `SPRING_DATASOURCE_*`") with the homelab contract (Postgres credentials synced from
  `kv/apps/<app>/postgres`). hl-common lets each Vault key map to any env var name, so no code
  has to assemble a URL from `POSTGRES_HOST`/`POSTGRES_PORT` (those are still injected and
  ignored). Flyway runs at startup with Postgres advisory locking (one replica anyway).
- **Postgres version**: the platform runs CloudNativePG `postgresql:17.7`; Testcontainers and
  `compose.yaml` pin `postgres:17.7-alpine` to match (the backend skill example uses 18 — the
  constitution says tests run on the production version, so 17 wins).

## R-008: Health and readiness endpoints

- **Decision — backend**: Spring Boot Actuator health probes, exposed on the main port at the
  homelab paths through additional paths:
  `management.endpoint.health.group.liveness.additional-path=server:/healthz` and
  `management.endpoint.health.group.readiness.additional-path=server:/readyz`, with the readiness
  group including `db`. `show-details: never`. Security permits `/healthz`, `/readyz` and the
  OpenAPI paths only. Hikari `connection-timeout` 2s and validation timeout 1s so a dead database
  flips readiness within one 5s probe period, well inside SC-007's 30 s.
- **Decision — frontend**: Route handlers `app/healthz/route.ts` and `app/readyz/route.ts`
  returning `200 {"status":"UP"}`, excluded from the proxy matcher. Readiness does not call the
  backend: the web app can still render the landing page and the "something went wrong" state
  without it, and a dependent readiness would cascade one outage into two.
- **Rationale**: Constitution Principle V and `service-contract.md` mandate `/healthz` and
  `/readyz`; the backend skill uses `/actuator/health/*`. Additional paths satisfy both without
  hand-written controllers. Auth0 is not a readiness dependency: JWKS is fetched lazily and
  cached.
- **Probes in the chart**: liveness `/healthz` every 10s, readiness `/readyz` every 5s, startup
  `/healthz` with `failureThreshold: 60, periodSeconds: 2` for the backend (JVM + Flyway start).

## R-009: Frontend authentication, session lifetime and sign-out

- **Decision**: `@auth0/nextjs-auth0` v4 (4.32.x) with the `Auth0Client` in `app/lib/auth0.ts`:
  - `authorizationParameters: { scope: 'openid profile email offline_access', audience }`.
  - `session: { rolling: true, inactivityDuration: 7 days, absoluteDuration: 30 days }` (FR-008).
  - Stateless encrypted HttpOnly `__session` cookie (`Secure`, `SameSite=Lax`); access and
    refresh tokens never cross into client components (FR-007, Principle III).
  - Auth routes `/auth/login`, `/auth/callback`, `/auth/logout` are served by the SDK from
    `proxy.ts`. They live **outside** `/api`, so the Traefik `/api` rule never captures them
    (the v3 SDK's `/api/auth/*` would have collided with the backend ingress).
  - `onCallback` hook: on an error (`access_denied`, cancelled or failed login) redirect to
    `/?signin=failed` (FR-006); on success redirect to `returnTo` only if it is a relative path
    on this origin, else `/home` (FR-003, US2-AS4). A unit test covers absolute, protocol-
    relative (`//evil.example`) and backslash variants.
  - Sign-out uses `/auth/logout`, which clears the cookie and ends the Auth0 session via OIDC
    RP-initiated logout, returning to `/` (FR-009). The return URL `APP_BASE_URL` is registered
    in Auth0's *Allowed Logout URLs*.
- **Auth0 tenant settings (operator, quickstart)**: Database connection with sign-ups disabled +
  Google social connection (FR-020); refresh-token rotation on, absolute lifetime 30 days,
  inactivity lifetime 7 days, aligned with the cookie; API with RS256 and the audience
  identifier; the Post-Login Action from R-003.
- **Known limit (documented, not a defect)**: after sign-out Auth0 asks for credentials again,
  but a user who chooses Google may be signed in by Google without retyping a password if their
  Google session is still alive. Auth0 cannot end the Google session.
- **Back button after sign-out (US3-AS4)**: protected pages are dynamic and read cookies, so Next
  serves them `Cache-Control: private, no-cache, no-store`; the proxy adds the same header to
  protected responses to be explicit. A Playwright spec checks it.

## R-010: Route protection and page model

- **Decision**: `proxy.ts` gates `PROTECTED_PREFIXES = ['/home']` and redirects to
  `/auth/login?returnTo=<path+query>`. The landing page `/` is public; when a session exists it
  shows "Go to HL Budget" linking to `/home` instead of "Sign in" (edge case). `app/not-found.tsx`
  is the global 404 (FR-022). The home page renders the profile in an async server component
  inside `<Suspense>` with a skeleton, calling `ensureMyProfile()` in `app/lib/data.ts` →
  `authedFetch('/api/v1/users/me', { method: 'PUT' })`. `app/home/error.tsx` is the "something
  went wrong" boundary with a "Try again" (`reset()` + `router.refresh()`) (FR-021). A 401 from
  the API redirects to `/auth/login?returnTo=/home` (FR-010); the proxy has already tried to
  refresh the access token.
- **Rationale**: The `frontend-developer` skill gates whole segments in the proxy to keep pages
  prerenderable, keeps per-user reads uncached and inside `<Suspense>`, and routes all API calls
  through `app/lib/api.ts`. `/home` is a dashboard placeholder that later features extend.
- **Server timeout (edge case "server slow but up")**: `apiFetch` uses
  `AbortSignal.timeout(API_TIMEOUT_MS)` (default 5000 ms); a timeout throws, the error boundary
  shows "something went wrong". Within SC-005's 2 s budget on the normal path.
- **Calling a mutating `PUT` during render**: allowed — it runs on the server, never during
  prerender (it reads cookies), and is idempotent (R-002), so React re-renders cannot duplicate
  records.

## R-011: Frontend → backend address

- **Decision**: `API_BASE_URL=http://hl-budget-backend` in the cluster (the sibling service on port
  80), `http://localhost:8080` locally. The frontend calls paths starting with `/api/v1/...`.
- **Rationale**: Principle II says the frontend calls the backend from the server. Going through
  the in-cluster service avoids a hairpin through Traefik, DNS and TLS, and works even when the
  ingress certificate is being changed. The backend still validates the bearer token.

## R-012: TLS certificate trusted on the private network (SC-004, US5-AS2)

- **Finding**: hl-helm today serves Traefik's built-in **self-signed** certificate
  (hl-helm research R-008, `docs/validation-results.md`), and plain HTTP returns 404 (accepted by
  US5-AS2: "redirected or refused"). The spec's assumption that the platform already provides a
  browser-trusted certificate is therefore **not yet true**.
- **Decision**: Use the hook hl-common already has: `ingress.tls.secretName`. The operator
  stores a certificate for the app's hostname in Vault at `kv/apps/hl-budget/tls`
  (keys `tls.crt`, `tls.key`); the chart syncs it with a `secrets.tls` entry of type
  `kubernetes.io/tls`, `keys: {}` (no env vars) and `rolloutRestart: false`, and both the `frontend`
  and `backend` ingresses reference `hl-budget-frontend-tls`. The certificate's source is the operator's
  choice and is outside the app repos: a Let's Encrypt certificate obtained by DNS-01 for a real
  domain whose record points at the private IP (trusted everywhere), or a private CA installed on
  the operator's devices.
- **Rationale**: Meets SC-004 without adding cert-manager to the platform (Principle VI) and
  without changing hl-common. It is a one-time operator step, not a per-release manual step.
- **Follow-up (platform, out of scope here)**: automatic renewal through cert-manager in hl-helm.
  Until then renewal is a Vault update every ≤90 days if Let's Encrypt is used; record this in
  the hl-helm PR body.
- **Alternatives considered**: Accept the self-signed certificate — fails SC-004. cert-manager now
  — infrastructure change to hl-helm, bigger than this feature. Deferred.

## R-013: Private-network-only exposure (FR-024)

- **Decision**: Nothing in the app or chart opens public access: Traefik listens on the node's
  LAN address, the router has no port-forward for it, and the hostname resolves only through the
  home network's DNS (and the VPN's DNS). The quickstart includes a check from outside the network
  (mobile data, VPN off) that the host does not resolve or does not connect.
- **Rationale**: Exposure is a network property the homelab already has; the spec puts it in
  deployment configuration. Adding an IP allow-list middleware would need a hl-common extension
  for no gain while Traefik is not internet-facing.

## R-014: Errors and the not-found contract

- **Decision — backend**: The backend skill's error package as-is: `GlobalExceptionHandler`,
  `ProblemDetailFactory`, `ProblemErrorController`, URNs `urn:problem:hl-budget:<category>`
  (`unauthenticated`, `access-denied`, `not-found`, `bad-request`, `internal-error`,
  `service-unavailable`), every response carrying `traceId` and `X-Request-Id`. A 5xx has a fixed
  safe detail. Unknown JSON properties are ignored.
- **Decision — frontend**: `error.tsx` per protected segment and `app/global-error.tsx`; errors
  are reported through `instrumentation.ts` `onRequestError` to stdout (no vendor yet,
  Principle VI) without headers. The UI never shows a status code, stack or `traceId` text
  except a short "Reference: <traceId>" line for support, which exposes no internals.
- **Rationale**: Constitution Principle II and FR-018 (one structured format, trace reference,
  no internals), SC-006.

## R-015: Containers

- **Decision — backend**: Multi-stage Dockerfile: `eclipse-temurin:21-jdk` (pinned minor +
  digest) builds with `./gradlew bootJar -x check`; runtime `eclipse-temurin:21-jre` pinned,
  `USER 10001:10001`, `EXPOSE 8080`, exec-form `ENTRYPOINT ["java","-XX:MaxRAMPercentage=75","-jar","/app/app.jar"]`,
  `JAVA_TOOL_OPTIONS=-Djava.io.tmpdir=/tmp`. tmpDirs `[/tmp]`. Memory limit 512Mi.
  `server.shutdown=graceful`, `spring.lifecycle.timeout-per-shutdown-phase=20s`.
- **Decision — frontend**: From `homelab` `templates/Dockerfile.node`, adapted to pnpm and Next
  `output: 'standalone'`; runtime `node:22-alpine` pinned, `USER 10001:10001`, `PORT=8080`,
  `HOSTNAME=0.0.0.0`, `CMD ["node","server.js"]` (no `pnpm start` wrapper, so SIGTERM reaches
  Node). tmpDirs `[/tmp, /app/.next/cache]`. Memory limit 256Mi.
- **Proof**: `hl-containerize` runs each image with `--read-only --tmpfs /tmp --user 10001:10001
  --cap-drop ALL --security-opt no-new-privileges` and checks `/healthz`, `/readyz` and
  `docker stop` time (< 10 s).
- **Architecture**: the cluster node's architecture is read by `hl-onboard-service`
  (`kubectl get nodes …architecture`) and written to `homelab.yaml` `platforms`.

## R-016: Testing strategy per spec success criterion

| Criterion | Where it is proven |
|---|---|
| SC-001, SC-005 | Playwright `e2e/sign-in.spec.ts` against a local stack with an Auth0 **test tenant** user; timings asserted loosely, measured on the deployed app in quickstart |
| SC-002 | Backend `UserProvisioningIT` (parallel PUTs, one row); Playwright two-context sign-in |
| SC-003 | Backend `SecurityIT` (every rejection path, no row created); Playwright `protected-routes.spec.ts` (signed out, expired cookie, API 401 → login) |
| SC-004 | `hl-deploy-status` after the hl-helm merge; quickstart deploy checks |
| SC-006 | Playwright `failures.spec.ts` (cancelled login via `?error=access_denied` callback, API down via a stopped backend / MSW, unknown URL) |
| SC-007 | Backend `ReadinessIT` (stop/start the Testcontainers Postgres, poll `/readyz`); repeated on the cluster in quickstart |
| SC-008 | Quickstart local section, run once on a clean checkout; gitleaks in CI on both repos |

Backend gate: `./gradlew check` (ktlint, detekt, tests, Kover 80/75 and 75 per package).
Frontend gate: `pnpm typecheck && pnpm lint && pnpm test`, plus `pnpm build` and Playwright in CI.
