# Contract: homelab deployment and configuration

**Feature**: `001-platform-foundation`. How both services plug into the homelab
(`homelab:*` skills, `hl-helm`). Values here are what `hl-onboard-service` writes; later
changes go through `hl-service-config`.

## Service link files

`hl-budget-backend/homelab.yaml`

```yaml
app: hl-budget
service: backend
image: hl-budget-backend
platforms: <node architecture, read by hl-onboard-service from kubectl>
```

`hl-budget-frontend/homelab.yaml`

```yaml
app: hl-budget
service: frontend
image: hl-budget-frontend
platforms: <same>
```

Each repo carries `.github/workflows/release.yaml` copied **unchanged** from the homelab plugin
template: push to `main` → image `ghcr.io/valyalkin/<image>:<run>-<sha7>` + digest → PR
`bump/hl-budget/<service>` in hl-helm. Merging that PR deploys (Argo CD, ~3 min). Images are
public on ghcr.

## Chart `hl-helm/charts/apps/hl-budget` (shape, not final values)

```yaml
frontend:
  workload:
    image: {name: hl-budget-frontend, tag: "<run>-<sha7>", digest: "sha256:<…>"}
    resources: {limits: {memory: 256Mi}}
    tmpDirs: [/tmp, /app/.next/cache]
    env:
      - {name: APP_BASE_URL, value: "https://<host>"}
      - {name: API_BASE_URL, value: "http://hl-budget-backend"}
      - {name: AUTH0_DOMAIN, value: "<tenant>.<region>.auth0.com"}
      - {name: AUTH0_CLIENT_ID, value: "<client id>"}        # public identifier, not a secret
      - {name: AUTH0_AUDIENCE, value: "<api identifier>"}
      - {name: API_TIMEOUT_MS, value: "5000"}
    probes:
      liveness:  {httpGet: {path: /healthz, port: http}, periodSeconds: 10}
      readiness: {httpGet: {path: /readyz,  port: http}, periodSeconds: 5}
      startup:   {httpGet: {path: /healthz, port: http}, failureThreshold: 30, periodSeconds: 2}
  service: {ports: [{name: http, port: 80, targetPort: 8080}]}
  ingress: {enabled: true, host: "<host>", path: /, tls: {secretName: hl-budget-frontend-tls}}
  secrets:
    auth0: {vaultPath: apps/hl-budget/frontend, keys: {auth0_client_secret: AUTH0_CLIENT_SECRET, auth0_secret: AUTH0_SECRET}}
    tls:   {vaultPath: apps/hl-budget/tls, type: kubernetes.io/tls, keys: {}, rolloutRestart: false}

backend:
  workload:
    image: {name: hl-budget-backend, tag: "<run>-<sha7>", digest: "sha256:<…>"}
    resources: {limits: {memory: 512Mi}}
    tmpDirs: [/tmp]
    env:
      - {name: SPRING_DATASOURCE_URL, value: "jdbc:postgresql://postgres-rw.postgres.svc.cluster.local:5432/hl_budget"}
      - {name: HLBUDGET_AUTH_ISSUER, value: "https://<tenant>.<region>.auth0.com/"}
      - {name: HLBUDGET_AUTH_AUDIENCE, value: "<api identifier>"}
      - {name: HLBUDGET_AUTH_JWKSETURI, value: "https://<tenant>.<region>.auth0.com/.well-known/jwks.json"}
      - {name: HLBUDGET_AUTH_CLAIMSNAMESPACE, value: "https://hl-budget/"}
      - {name: LOGGING_LEVEL_ROOT, value: INFO}
      - {name: SPRINGDOC_API_DOCS_ENABLED, value: "false"}     # no public API document on the shared host
      - {name: SPRINGDOC_SWAGGER_UI_ENABLED, value: "false"}
    probes:
      liveness:  {httpGet: {path: /healthz, port: http}, periodSeconds: 10}
      readiness: {httpGet: {path: /readyz,  port: http}, periodSeconds: 5}
      startup:   {httpGet: {path: /healthz, port: http}, failureThreshold: 60, periodSeconds: 2}
  service: {ports: [{name: http, port: 80, targetPort: 8080}]}
  ingress: {enabled: true, host: "<host>", path: /api, tls: {secretName: hl-budget-frontend-tls}}
  dependencies:
    postgres:
      host: postgres-rw.postgres.svc.cluster.local
      port: 5432
      credentials: {vaultPath: apps/hl-budget/postgres, keys: {username: SPRING_DATASOURCE_USERNAME, password: SPRING_DATASOURCE_PASSWORD}}
```

Plus `bootstrap/values/postgres.yaml`: `- {name: hl_budget, namespace: hl-budget}` (Postgres names cannot contain a hyphen; the credentials still land at `kv/apps/hl-budget/postgres`), and
`ci/default-values.yaml` with `hl-budget.ci.example` for both ingresses.

## Environment variables

### hl-budget-backend (`.env.example`; typed in `HlBudgetProperties`, `@Validated`, fails fast)

| Variable | Secret | Required | Example (placeholder) | Purpose |
|---|---|---|---|---|
| `PORT` | no | no (8080) | `8080` | Listening port (`server.port: ${PORT:8080}`) |
| `SPRING_DATASOURCE_URL` | no | yes in cluster | `jdbc:postgresql://localhost:5432/hl-budget` | Not set locally (docker compose provides it) |
| `SPRING_DATASOURCE_USERNAME` | **yes** (Vault) | yes in cluster | `changeme` | From `kv/apps/hl-budget/postgres` |
| `SPRING_DATASOURCE_PASSWORD` | **yes** (Vault) | yes in cluster | `changeme` | From `kv/apps/hl-budget/postgres` |
| `HLBUDGET_AUTH_ISSUER` | no | yes | `https://your-tenant.eu.auth0.com/` | Expected `iss` (trailing slash) |
| `HLBUDGET_AUTH_AUDIENCE` | no | yes | `https://api.hl-budget.example` | Expected `aud` |
| `HLBUDGET_AUTH_JWKSETURI` | no | yes | `https://your-tenant.eu.auth0.com/.well-known/jwks.json` | Signing keys |
| `HLBUDGET_AUTH_CLAIMSNAMESPACE` | no | yes | `https://hl-budget/` | Prefix of the profile claims |

### hl-budget-frontend (`.env.example`; validated by Zod in `app/lib/env.ts`, fails fast)

| Variable | Secret | Required | Example (placeholder) | Purpose |
|---|---|---|---|---|
| `PORT` | no | no (8080 in image; 3000 in `pnpm dev`) | `8080` | Listening port |
| `APP_BASE_URL` | no | yes | `http://localhost:3000` | Public origin; callback and logout return |
| `API_BASE_URL` | no | yes | `http://localhost:8080` | Backend origin, server-side only |
| `API_TIMEOUT_MS` | no | no (5000) | `5000` | Abort slow backend calls |
| `AUTH0_DOMAIN` | no | yes | `your-tenant.eu.auth0.com` | Tenant |
| `AUTH0_CLIENT_ID` | no | yes | `your-client-id` | Regular Web Application |
| `AUTH0_CLIENT_SECRET` | **yes** (Vault) | yes | `changeme` | From `kv/apps/hl-budget/frontend` key `auth0_client_secret` |
| `AUTH0_SECRET` | **yes** (Vault) | yes | `generate with: openssl rand -hex 32` | Session encryption key, `kv/apps/hl-budget/frontend` key `auth0_secret` |
| `AUTH0_AUDIENCE` | no | yes | `https://api.hl-budget.example` | Must equal the backend's audience |

No `NEXT_PUBLIC_*` variables exist in this feature.

## Vault paths the operator fills (never in Git)

| Path | Keys | Notes |
|---|---|---|
| `kv/apps/hl-budget/frontend` | `auth0_client_secret`, `auth0_secret` | Rotating `auth0_secret` signs everyone out |
| `kv/apps/hl-budget/tls` | `tls.crt`, `tls.key` | Certificate for `<host>` (research R-012) |
| `kv/apps/hl-budget/postgres` | `username`, `password` | **Generated** by `bootstrap/install.sh --only postgres`; do not create by hand |

## Hardened runtime (both images, proven by `hl-containerize`)

UID/GID 10001, non-root, read-only root filesystem, writes only to `tmpDirs`, all capabilities
dropped, no privilege escalation, port 8080, logs to stdout, SIGTERM → exit within 30 s, no
secrets in layers or build args.
