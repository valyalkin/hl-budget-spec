# Quickstart & validation: Platform Foundation

**Feature**: `001-platform-foundation` | **Plan**: [plan.md](./plan.md)

A run guide that proves the feature end to end: locally first (US6, SC-008), then deployed
(US5, SC-004, SC-007). Configuration is in [contracts/deployment.md](./contracts/deployment.md);
the API in [contracts/openapi.yaml](./contracts/openapi.yaml); pages in
[contracts/web-routes.md](./contracts/web-routes.md).

## Prerequisites

- Docker (for Postgres, Testcontainers and the container checks), JDK 21, Node 22 + pnpm 10.
- `gh` logged in as `valyalkin`; `helm` 4.x, `helm-docs`, `kubeconform` for hl-helm.
- Worktrees `hl-budget-backend/` and `hl-budget-frontend/` in this repo on branch
  `001-platform-foundation` (research R-001), and `~/Documents/Projects/hl-helm`.
- Two Auth0 tenants or applications: a **dev/test** one (local + Playwright, seeded test user,
  never real users) and **prod**.

## 1. Auth0 setup (once per tenant, operator)

Done by hand in the Auth0 Dashboard, about 15 minutes per tenant. `<APP_BASE_URL>` is
`http://localhost:3000` for the dev/test tenant and the public `https://` origin for prod. No
secret is ever shared or committed: the client secret and session key go only into `.env.local`
(dev) or Vault (prod).

### 1.0 Create the tenant

*What it does*: gives this environment its own isolated set of users, keys and settings, so dev
sign-ins and test users never touch prod.

1. At manage.auth0.com, create a tenant, e.g. `hl-budget-dev` (environment tag **Development**)
   or `hl-budget` (**Production**).
2. Note the tenant **domain**, e.g. `hl-budget-dev.eu.auth0.com`.

### 1.1 API (the backend's audience)

*What it does*: registers the backend as a protected resource. Auth0 then issues access tokens
whose `aud` is this identifier, signed with the tenant's RS256 keys; the backend accepts only
those (`HLBUDGET_AUTH_AUDIENCE`, `HLBUDGET_AUTH_JWKSETURI`).

1. **Applications → APIs → Create API**: Name `HL Budget API`; **Identifier**
   `https://api.hl-budget.example` (the audience; it cannot be changed later); signing algorithm
   **RS256**.
2. In the API's **Settings**, turn on **Allow Offline Access**. The frontend asks for
   `offline_access`; without this Auth0 issues no refresh token and sessions cannot roll.
3. After creating the application (1.2), come back to the API's **Application Access** tab and
   set `HL Budget Web`'s **User Access** to **Authorized** (or the API's default user-access
   policy to Allow). *Newer tenants deny an application tokens for an API until it is
   authorized.* Without it every sign-in fails with "Client … is not authorized to access
   resource server …".

### 1.2 Application (the frontend)

*What it does*: registers the web app as a confidential OIDC client. The allowed URLs are the
only places Auth0 will send users back to after sign-in and sign-out, and the refresh-token
settings bound how long a session can live (7 days idle, 30 days at most).

1. **Applications → Applications → Create Application**: Name `HL Budget Web`, type **Regular
   Web Applications**.
2. **Settings**:
   - **Allowed Callback URLs**: `<APP_BASE_URL>/auth/callback`
   - **Allowed Logout URLs**: `<APP_BASE_URL>`
   - **Allowed Web Origins**: `<APP_BASE_URL>`
3. **Refresh Token Rotation**: on. *Each refresh issues a new token and invalidates the old one,
   so a stolen refresh token stops working.*
4. **Refresh Token Expiration**: **Absolute Expiration** on, lifetime `2592000` (30 d);
   **Inactivity Expiration** on, lifetime `604800` (7 d).
5. **Save**. Note the **Client ID** (not secret). The **Client Secret** is secret.

### 1.3 Connections

*What it does*: decides who can sign in and how. Disabling sign-ups makes this an invite-only
app: only accounts the operator creates can get in.

1. **Authentication → Database → Username-Password-Authentication**: turn on **Disable Sign
   Ups**; on its **Applications** tab enable only `HL Budget Web`.
2. **Authentication → Social → Google**: create it if missing (Auth0's dev keys are fine for the
   dev tenant; prod needs your own Google OAuth client); on its **Applications** tab enable only
   `HL Budget Web`.
3. On `HL Budget Web` → **Connections**, confirm only these two are on.

### 1.4 Profile-claims Action

*What it does*: an access token for an API carries no name, email or picture. This Action adds
them as namespaced claims (`https://hl-budget/{name,email,picture}`), signed with the token, so
the backend can keep the user's record current (research R-003). The namespace must equal
`HLBUDGET_AUTH_CLAIMSNAMESPACE`.

1. **Actions → Library → Create Action → Build from scratch**: Name `HL Budget profile claims`,
   Trigger **Login / Post Login**, Runtime **Node 22**.
2. Replace the editor contents with `hl-budget-backend/auth0/post-login-profile-claims.js` and
   **Deploy**. That file is the source of truth; change it there first, then paste again.
3. **Actions → Triggers → post-login**: drag the Action between Start and Complete, **Apply**.

### 1.5 Users

*What it does*: since sign-ups are off, creates the accounts that may sign in. On the dev
tenant this is the Playwright test user.

1. **User Management → Users → Create User**: connection Username-Password-Authentication, an
   email (dev: e.g. `e2e@hl-budget.test`) and a strong password.
2. Optional: set the user's **name** so `/home` greets by name (otherwise it falls back to email).
3. Caveat: **Disable Sign Ups** applies only to the database connection. Through Google, any
   Google account can sign in and gets a profile on first use. If that is not intended (prod),
   restrict it, e.g. with an allow-list check in a Post-Login Action that calls `api.access.deny`.

### 1.6 Wire the values in

*What it does*: points both services at this tenant. Only the client secret and session key are
secret.

| Variable | Service | Value | Secret |
|---|---|---|---|
| `HLBUDGET_AUTH_ISSUER` | backend | `https://<domain>/` (trailing slash required) | no |
| `HLBUDGET_AUTH_JWKSETURI` | backend | `https://<domain>/.well-known/jwks.json` | no |
| `HLBUDGET_AUTH_AUDIENCE` / `AUTH0_AUDIENCE` | both | `https://api.hl-budget.example` | no |
| `HLBUDGET_AUTH_CLAIMSNAMESPACE` | backend | `https://hl-budget/` | no |
| `AUTH0_DOMAIN` | frontend | `<domain>` | no |
| `AUTH0_CLIENT_ID` | frontend | Client ID from 1.2 | no |
| `AUTH0_CLIENT_SECRET` | frontend | Client Secret from 1.2 | **yes** |
| `AUTH0_SECRET` | frontend | output of `openssl rand -hex 32` | **yes** |

Dev: put them in each repo's `.env.local` (§2). Prod: chart values plus Vault
`kv/apps/hl-budget/frontend`. For Playwright, export the dev test user in the shell, never in a
file: `export E2E_USERNAME='…' E2E_PASSWORD='…'` (CI: repository secrets).

Expected: with both services running (§2), **Sign in** opens Universal Login offering
email/password and Google, with no "Sign up" link; signing in lands on `/home` greeting the user.

## 2. Run locally (US6, SC-008)

```sh
# backend: starts its own Postgres 17 via docker compose (random host port)
cd hl-budget-backend    # from the hl-budget-spec root
cp .env.example .env.local   # fill HLBUDGET_AUTH_* from the dev tenant
./gradlew bootRun --args='--spring.profiles.active=local'

# frontend
cd ../hl-budget-frontend
cp .env.example .env.local   # fill AUTH0_* from the dev tenant; AUTH0_SECRET=$(openssl rand -hex 32)
pnpm install && pnpm dev     # http://localhost:3000
```

Checks:

| # | Action | Expected |
|---|---|---|
| L1 | `curl -s localhost:8080/healthz` and `/readyz` | `{"status":"UP"}` both |
| L2 | `curl -si -X PUT localhost:8080/api/v1/users/me` | `401`, `application/problem+json`, `type` `urn:problem:hl-budget:unauthenticated`, `traceId` equals `X-Request-Id` |
| L3 | Open `http://localhost:3000` | Landing with "HL Budget", description, **Sign in** |
| L4 | Sign in with the dev user | `/home` greets you by name with email and avatar |
| L5 | `docker exec` into the compose Postgres: `select count(*) from users where auth0_subject = '<sub>'` | `1`; sign in again → still `1`, `last_seen_at` advanced |
| L6 | Remove `HLBUDGET_AUTH_AUDIENCE` and restart the backend; remove `AUTH0_SECRET` and restart the web app | Each fails at startup naming the missing setting |
| L7 | `git grep -nE '(secret|password|token)\s*[:=]\s*[^<c$]' -- . ':!*.md'` in both repos; gitleaks in CI | No real secrets |

## 3. Quality gates (must be green before merge, Principle IV)

```sh
cd hl-budget-backend && ./gradlew check                  # ktlint, detekt, unit + Testcontainers IT, Kover gates
cd hl-budget-frontend && pnpm typecheck && pnpm lint && pnpm test && pnpm build
cd hl-budget-frontend && pnpm generate:api && git diff --exit-code app/lib/api-schema.d.ts   # contract not stale
cd hl-budget-frontend && pnpm e2e                          # Playwright, against the dev tenant test user
```

Key tests to see pass: `SecurityIT`, `UserProvisioningIT` (parallel first calls → one row),
`ReadinessIT` (Postgres stop/start), `GlobalExceptionHandlerIT`, `OpenApiExportIT`; Playwright
`sign-in`, `protected-routes`, `sign-out`, `failures`, `not-found`.

## 4. Container proof (`hl-containerize`, per repo)

Run the `hl-containerize` skill in each repo. Expected report: port 8080; `/healthz` 200;
`/readyz` 200 for web and 503 for the API without a database (correct); `docker stop` < 10 s;
writable dirs `/tmp` (api) and `/tmp`, `/app/.next/cache` (web); no `.env` in the image.

## 5. Deploy (`hl-onboard-service`, then `hl-deploy-status`)

1. In `hl-budget-backend`: run **hl-onboard-service** ("deploy this to the homelab"). Creates the
   GitHub repo, `homelab.yaml`, release workflow, first image, and the hl-helm PR creating
   `charts/apps/hl-budget` with the `backend` block and the Postgres database.
2. In `hl-budget-frontend`: run **hl-onboard-service** again; it extends the same app with the `frontend`
   block on the same host (`/`), next to the API's `/api`.
3. Operator, in order: flip both ghcr packages to public; Vault `kv/apps/hl-budget/frontend` and
   `kv/apps/hl-budget/tls`; merge the hl-helm PR(s);
   `bootstrap/install.sh --only postgres --diff` then apply; internal DNS record for `<host>`
   (home network + VPN resolver); set `HL_HELM_TOKEN` in both repos.
4. Run **hl-deploy-status** ("did it deploy?") for `hl-budget/backend` and `hl-budget/frontend`.

Deployed checks (SC-004 within 10 min of merging a release PR, 0 manual steps after onboarding):

| # | Action | Expected |
|---|---|---|
| D1 | Argo CD app `hl-budget` | `Synced` / `Healthy`; both pods Ready |
| D2 | Open `https://<host>/` from the home network and over VPN | Landing page, browser shows a valid certificate |
| D3 | `curl -sI http://<host>/` | Refused or 404 (no plain-HTTP content) |
| D4 | From mobile data with VPN off | Host does not resolve / connect |
| D5 | Sign in with a prod account; reload `/home` | Profile in < 2 s (SC-005) |
| D6 | SC-007 on the local stack (the cluster Postgres is shared, so don't stop it): `docker stop` the compose Postgres, poll `curl -s -o /dev/null -w '%{http_code}' localhost:8080/readyz`, then `docker start` it | `/readyz` 503 within 30 s while `/healthz` stays 200; 200 again within 30 s of restart. Automated by `ReadinessIT` |
| D7 | Push a trivial commit to `main` of either repo, merge the bump PR | New version rolls out and reports healthy with no other action |

### Results, 2026-10-26 (dev Auth0 tenant, `hl-budget.home.example`)

| Check | Result |
|---|---|
| D1 | Pass: `Synced`/`Healthy` at hl-helm `main`; backend `2-548d0ff`, frontend `2-bb4367e` Ready |
| D2 | Partial: works from the home network via `/etc/hosts`; certificate is Traefik's self-signed one and VPN is not set up (deferred by the operator) |
| D3 | Pass: plain HTTP answers 404 |
| D4 | Pass by construction: `.example` does not resolve in public DNS |
| D5 | Pass (operator): signed in on the deployed app and saw the profile |
| D6 | Pass: `ReadinessIT` in `./gradlew check` |
| D7 | Pass: merges to both `main`s opened hl-helm #11 and #12 on their own; merged, Argo CD rolled out backend `4-ba06a1a` and frontend `5-7e9a7e1`, `Healthy` (the token first lacked Contents write on hl-helm: 403 on push) |

§6 on the cluster (`pnpm e2e` with `APP_BASE_URL=https://hl-budget.home.example`): 14 passed,
1 skipped. The skip is the API-down journey (US4-AS4/5), which needs a web server pointed at a
closed API port (`E2E_DOWN_BASE_URL`); it was not exercised on the cluster.

## 6. Journey checks (manual, on the deployed app)

Most of these are covered by the Playwright suite, which can run against the cluster. It needs
the `/etc/hosts` entry (or DNS) for the host and the test user's credentials in your shell:

```sh
cd hl-budget-frontend
APP_BASE_URL=https://hl-budget.home.example E2E_IGNORE_HTTPS_ERRORS=1 pnpm e2e
```

`E2E_IGNORE_HTTPS_ERRORS=1` is only for the self-signed certificate; drop it once a trusted one
is in Vault. Nothing is started locally, because the deployed `/healthz` already answers.

| Story | Steps | Expected |
|---|---|---|
| US2 | Signed out, open `https://<host>/home?x=1` | Login, then back on `/home?x=1` |
| US2-AS4 | `https://<host>/auth/login?returnTo=https://example.com` | After login, on `/home` |
| US3 | Sign out; press Back; open `/home`; click Sign in | Landing; no profile shown; login asked; credentials requested |
| US4-AS1 | Cancel on the login page | Landing with "Sign-in did not complete" |
| US4-AS4/5 | Scale the API to 0 via hl-service-config **only on a test branch**, or stop the local backend; load `/home`; restore; **Try again** | "Something went wrong" with Try again, then the profile |
| US4-AS6 | Open `/nope` | Not-found page with a link to `/` |
