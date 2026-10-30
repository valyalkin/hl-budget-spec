# Research: Home and Remote Access

**Feature**: [spec.md](spec.md) | **Plan**: [plan.md](plan.md) | **Date**: 2026-10-30

Facts about the running homelab were read from the cluster (`KUBECONFIG=~/.kube/homelab.yaml`)
and from `valyalkin/hl-helm` `main` (`ab37b72`) on 2026-10-30:

- One k3s node `alex-thinkcentre-m710q`, `10.0.0.100`, k3s v1.36.5. Traefik 3.7.13 from the
  k3s-bundled chart `traefik-40.1.5+up40.1.0`, Service `LoadBalancer` with external IP
  `10.0.0.100` (ports 80, 443). No `HelmChartConfig`, no `TLSStore`, no cert-manager, no
  Tailscale. Traefik CRDs (`tlsstores.traefik.io` …) are installed.
- hl-helm tiers: bootstrap and infrastructure charts are applied by `bootstrap/install.sh` in the
  order of `bootstrap/install-order.yaml`; Argo CD manages `charts/apps/*` only.
- `hl-common.ingress` always emits a TLS block; when `tls.secretName` is unset, Traefik serves its
  **default certificate** (hl-helm research R-008). Ingresses listen on `websecure` only.
- Vault: Kubernetes auth with roles `apps` (any namespace, reads `kv/apps/<own ns>/*`),
  `infra` (bound to `postgres-vault`/`redis` only, reads `kv/infra/*`) and `provision`.
  Infrastructure charts bind with `hl-common.namespace` (`vaultAuth.role`, `createNamespace: false`).
- No hostname is hard-coded in `hl-budget-frontend` or `hl-budget-backend`; the address lives in
  `charts/apps/hl-budget/values.yaml` (both ingress hosts and `APP_BASE_URL`).
- Latest stable versions on 2026-10-30: cert-manager **v1.21.3** (chart
  `oci://quay.io/jetstack/charts/cert-manager`), Tailscale **v1.104.4**.

---

## R-001 Address scheme

- **Decision**: Flat names directly under the domain: `budget.<domain>` for HL Budget,
  `nginx.<domain>` for the reference app, the bare `<domain>` reserved for a future personal page.
- **Rationale**: A wildcard certificate covers exactly one label level, so flat names keep one
  certificate valid for all apps (FR-003). The same names can later be published without renaming
  (spec, "Going public later").
- **Alternatives considered**: `budget.home.<domain>` (needs a second wildcard, `*.home.<domain>`,
  and a rename when going public); keeping `hl-budget` as the label (longer; the owner agreed on
  `budget`).

## R-002 DNS records

- **Decision**: One record in Cloudflare, created by hand: `*.<domain>  A  10.0.0.100`,
  **DNS only** (grey cloud), TTL Auto. No apex record until a personal page exists. No
  in-cluster DNS automation.
- **Rationale**: A wildcard record means a new app needs no DNS step at all, which beats SC-004.
  A specific record (for example a later public `budget.<domain>` pointing at a tunnel) overrides
  the wildcard, so going public stays possible. Cloudflare cannot proxy a private address, so the
  record must be DNS-only.
- **Alternatives considered**: one record per app (an extra manual step each time); ExternalDNS
  (another controller and another credential with write access, for a record that changes almost
  never; hl-helm Principle VII).

## R-003 Certificate issuance

- **Decision**: cert-manager **v1.21.3**, ACME with Let's Encrypt, **DNS-01 via the Cloudflare
  API**. One `Certificate` for `<domain>` and `*.<domain>`. A namespaced `Issuer` (not a
  `ClusterIssuer`) in the same namespace as the certificate. Start on the Let's Encrypt
  **staging** server, switch to production once the staging certificate is issued (a values
  switch). Default renewal (two thirds of the lifetime; about 30 days before expiry for a 90-day
  certificate, about 15 days for a 45-day one).
- **Rationale**: DNS-01 is the only ACME challenge that issues wildcards and needs nothing
  reachable from the internet (FR-004, FR-010). A namespaced `Issuer` keeps the Cloudflare token
  in one namespace instead of the cert-manager namespace, and there is only one certificate to
  issue. Staging first avoids Let's Encrypt rate limits while debugging.
- **Alternatives considered**: Traefik's built-in ACME resolver (keeps certificates in a file or
  volume inside Traefik, needs the token in `kube-system` and the k3s-managed Traefik release;
  harder to inspect and to reuse); `tailscale cert` / `*.ts.net` names (rejected in the
  discussion: no own domain, every device needs the VPN); mkcert / private CA (declined by the
  owner: per-device setup).

## R-004 Sharing the certificate with every app

- **Decision**: Make the wildcard certificate Traefik's **default certificate** through a Traefik
  `TLSStore` named `default`, in the same namespace as the certificate Secret. Apps leave
  `ingress.tls.secretName` unset (hl-common already supports this).
- **Rationale**: New apps need no certificate settings, Secrets or Vault entries (FR-003, Story 3).
  No copy of the private key is needed in other namespaces.
- **Alternatives considered**: a `Certificate` per app namespace from a `ClusterIssuer` (a
  certificate request and a Cloudflare credential reachable per app, more Let's Encrypt orders);
  copying the Secret into each namespace with a reflector/replicator (another controller, key
  copies everywhere).
- **To verify during implementation** (k3d test cluster first): Traefik 3.7 picks up a `TLSStore`
  named `default` outside `kube-system`. If it does not, the store and certificate move to
  `kube-system`; nothing else changes.
- **Confirmed 2026-10-30 (T018, k3d, Traefik chart 39.0.7)**: a `TLSStore` named `default` in
  namespace `ingress-tls` is honoured. With SNI `nginx.valyalkin.com`, Traefik presented the
  staging certificate (`CN=valyalkin.com`, SAN `*.valyalkin.com, valyalkin.com`). No
  `kube-system` fallback is needed. The staging certificate was issued about 2.5 minutes after
  the solver started (DNS-01 via Cloudflare, public-resolver self-check). Let's Encrypt now
  issues **64-day** certificates; cert-manager renews at two thirds of the lifetime, which is
  about 21 days before expiry, so FR-004's "at least 2 weeks" still holds.

## R-005 HTTP to HTTPS redirect

- **Decision**: A k3s `HelmChartConfig` named `traefik` in `kube-system` that sets a permanent
  redirect from the `web` entry point to `websecure` (FR-006). The exact key for chart 40.x
  (`ports.web.http.redirections.entryPoint` or the older `ports.web.redirections.entryPoint`) is
  confirmed against the bundled chart's `values.yaml` before applying.
- **Rationale**: This is the k3s-supported way to configure the built-in Traefik, and one setting
  covers every app. It is recorded in hl-helm, so a cluster rebuild reproduces it.
- **Alternatives considered**: a redirect `Middleware` plus a `web` router per app (repeats in every
  chart); leaving port 80 returning 404 (fails FR-006).
- **Confirmed 2026-10-30 (T008)**: the bundled chart `traefik-40.1.5+up40.1.0` uses
  `ports.web.http.redirections.entryPoint` (default `{}`), with fields `to`, `scheme`, `permanent`
  and `priority`. The `HelmChartConfig` sets
  `ports.web.http.redirections.entryPoint: {to: websecure, scheme: https, permanent: true}`. On k3d the chart turned this into
  `--entryPoints.web.http.redirections.entryPoint.to=:443 …permanent=true`, and
  `http://nginx.valyalkin.com/` answered `308 Permanent Redirect` to `https://…` (T018).

## R-006 Remote access (VPN)

- **Decision**: **Tailscale**, with a small in-cluster **subnet router**: one pod running the
  official `tailscale/tailscale:v1.104.4` image in **userspace networking** mode, advertising the
  single route `10.0.0.100/32`, tagged `tag:homelab`. The pod keeps its node state in a Kubernetes
  Secret, so restarts do not re-register. It authenticates with a **Tailscale OAuth client
  secret** used as an auth key (`?ephemeral=false&preauthorized=true`) with the tag, which does not
  expire, and tagged devices have key expiry disabled (FR-013).
- **Rationale**:
  - Outbound-only, no router port forwarding (FR-010). Free for one person. Apps for iOS, Android,
    macOS and Windows.
  - Advertising only `10.0.0.100/32`, plus an access rule allowing only ports 80 and 443, gives
    VPN devices the homelab address and nothing else (FR-011).
  - Userspace mode needs no `NET_ADMIN`, no privileged init and no host networking, so the pod
    can run non-root with a read-only root filesystem like every other homelab workload.
  - From inside the cluster, traffic to `10.0.0.100:443` reaches Traefik: kube-proxy handles the
    LoadBalancer IP for pod traffic.
- **Alternatives considered**:
  - Tailscale Kubernetes operator with a `Connector` (operator, CRDs and proxy pods; the proxies
    run privileged by default; more than a single route needs; can be added later to publish
    services directly).
  - Tailscale installed on the node's OS (simplest, but outside hl-helm, so a rebuild from the
    repository would not restore it; hl-helm Principle I).
  - Self-hosted WireGuard / router VPN (needs port forwarding and per-device keys).
  - Cloudflare Tunnel (public exposure, not a VPN; reserved for going public later).
- **To verify during implementation**: subnet routing in userspace mode as a non-root user with
  a read-only root filesystem (`TS_USERSPACE=true`, `TS_KUBE_SECRET`, `TS_SOCKET` and state under
  a tmp dir); the health endpoint (`TS_ENABLE_HEALTH_CHECK`, `TS_LOCAL_ADDR_PORT`) for probes.

## R-007 Tailscale access policy

- **Decision**: The tailnet policy file (edited by the owner in the Tailscale admin console, a
  manual step) contains:
  - a tag owner for `tag:homelab` (the owner);
  - `autoApprovers.routes` approving `10.0.0.100/32` for `tag:homelab`, so the route is live
    without a manual click;
  - one access rule letting the owner's own devices reach `10.0.0.100:80,443` only, and nothing
    allowing `tag:homelab` to start connections to the owner's devices.

  The exact text is in [contracts/tailnet-policy.md](contracts/tailnet-policy.md).
- **Rationale**: Least privilege for FR-011 and FR-012; a lost device is removed in one place in
  the admin console.
- **Alternatives considered**: the default allow-all policy (VPN devices could reach every port
  of the node, including SSH and the Kubernetes API).

## R-008 Secrets

- **Decision**: Two new Vault entries under the existing infrastructure prefix:
  `kv/infra/cloudflare` (`api_token`) and `kv/infra/tailscale` (`client_secret`). A new Vault
  Kubernetes role **`edge`**, bound to the service accounts of the two new charts, whose policy
  reads only those two paths. `bootstrap/vault-init.sh` gains the role and policy (idempotent,
  re-run once). VSO syncs each value into its chart's namespace.
- **Rationale**: Secrets only from Vault (constitution III; hl-helm Principle V). A separate role
  avoids widening the `infra` role, which can read every `kv/infra/*` entry, including Redis.
- **Alternatives considered**: reusing the `infra` role (each new chart could read the other's
  credential and Redis's); Kubernetes Secrets created by hand (not reproducible, not in Vault).

## R-009 Where the new pieces live in hl-helm

- **Decision**: Three new **infrastructure-tier** charts, applied by `bootstrap/install.sh` after
  `redis` and before Argo CD:
  1. `charts/infrastructure/cert-manager`: wraps the upstream chart v1.21.3, with CRDs.
  2. `charts/infrastructure/ingress-tls`: the Cloudflare token sync, the `Issuer`, the wildcard
     `Certificate`, the default `TLSStore` and the Traefik `HelmChartConfig`.
  3. `charts/infrastructure/tailscale`: the subnet router.

  cert-manager and ingress-tls are separate releases because a release cannot create the
  cert-manager CRDs and use them in the same install.
- **Rationale**: The certificate and VPN serve every app, so they are platform pieces, not part of
  any app chart (FR-016). Infrastructure is applied by the operator after a `--diff` review
  (hl-helm Principle III), so a merge can never break ingress for every app at once.
- **Alternatives considered**: putting them in `charts/apps/` under Argo CD (a merge would
  reconfigure ingress for every app with no apply-time review, and Argo CD would manage
  cluster-wide CRDs); putting them in the hl-budget chart (FR-016, one app per chart).

## R-010 HL Budget and nginx changes

- **Decision**: In `charts/apps/hl-budget/values.yaml`, set both ingress hosts to `budget.<domain>`
  and `APP_BASE_URL` to `https://budget.<domain>`. Remove `ingress.tls.secretName` from both
  services and `frontend.secrets.tls` (the empty Vault path `kv/apps/hl-budget/tls` is deleted).
  Move `nginx` to `nginx.<domain>` the same way and update the reference-chart guard
  (`hl-common/placeholders`) in both charts, the policy self-test and `docs/`. No change to
  `hl-budget-frontend` or `hl-budget-backend` code or images.
- **Rationale**: FR-001, FR-015, FR-018 and Story 3. The address already lives only in the values
  file.
- **Alternatives considered**: keeping the old host as a second ingress rule during the switch
  (Auth0 would have to accept both; the switch is a single merge, and a rollback is reverting it).

## R-011 Sign-in provider

- **Decision**: The owner updates the existing Auth0 application by hand (a manual task). Replace
  `https://hl-budget.home.example` with `https://budget.<domain>` in Allowed Callback URLs
  (`/auth/callback`), Allowed Logout URLs and Allowed Web Origins, **at the same time as** the
  hl-budget values PR is merged. The API audience identifier `https://api.hl-budget.example` is a
  name, not an address, and stays.
- **Rationale**: FR-014. The Auth0 tenant is configured by hand in feature 001's quickstart; there
  is no Auth0-as-code in this project.
- **Alternatives considered**: Auth0 Deploy CLI / Terraform (worth it with several apps or
  tenants, not for one change).

## R-012 Visibility of certificate health

- **Decision**: Extend `scripts/infra-status.sh` to print the wildcard certificate's `Ready`
  condition and expiry (`status.notAfter`), the Tailscale router pod's state, and to include the
  three new namespaces in its unhealthy-pod check. cert-manager also records renewal failures as
  events and a `Ready=False` condition on the `Certificate`.
- **Rationale**: FR-005. Argo CD does not watch infrastructure, and `infra-status.sh` is already
  the report the owner runs for that tier.
- **Alternatives considered**: Prometheus alerts on cert-manager metrics (there is no monitoring
  stack in the homelab yet; Principle VI).

## R-013 Private-address DNS answers at home (router rebind protection)

- **Decision**: The quickstart gives a two-command check from a laptop on the home Wi-Fi: resolve
  `budget.<domain>` through the router, then through `1.1.1.1`. If only the second returns
  `10.0.0.100`, the router filters private answers. Fixes, in order: a rebind exception for
  `<domain>` in the router; otherwise keep Tailscale on at home on that device (Tailscale's DNS
  resolver returns the answer, and the subnet route reaches the node).
- **Rationale**: FR-020; the router model is unknown (spec Clarifications).
- **Alternatives considered**: running a DNS server in the homelab (Pi-hole/AdGuard) and pointing
  the router's DHCP at it (more to run and a single point of failure for the household's DNS;
  can be done later if needed).

## R-014 Test approach

- **Decision**:
  - hl-helm's existing gate for chart changes: `scripts/validate.sh` (lint, template, kubeconform,
    helm-unittest, policy tests), with unit tests for the new charts.
  - A rehearsal on the local k3d test cluster for cert-manager, the TLSStore and the redirect,
    using the Let's Encrypt staging server.
  - `bootstrap/install.sh --only <name> --diff` review before each infrastructure apply.
  - The quickstart's live checks for the success criteria: devices at home, VPN on and off,
    port scan, nginx move, forced renewal.

  The service repos' gates are untouched because their code does not change.
- **Rationale**: Constitution IV covers service code; for platform charts hl-helm's own
  Principle III gate applies.
