# Implementation Plan: Home and Remote Access

**Branch**: `005-home-and-remote-access` | **Date**: 2026-10-30 | **Spec**: [spec.md](spec.md)

**Input**: Feature specification from `/specs/005-home-and-remote-access/spec.md`

## Summary

Give HL Budget one stable, trusted address, `https://budget.<domain>`, that works on any device
at home and, over Tailscale, away from home, with no ports open to the internet.

The work is platform configuration in `valyalkin/hl-helm`, plus manual account steps for the
owner. There are no code changes in the two service repositories.

- **Domain and DNS**: a domain on Cloudflare with one wildcard record, `*.<domain> → 10.0.0.100`
  (DNS only).
- **Certificate**: cert-manager v1.21.3 gets one Let's Encrypt certificate for `<domain>` and
  `*.<domain>`, proving ownership through the Cloudflare DNS API (DNS-01).
- **Sharing it**: the certificate becomes Traefik's default certificate, so every app without its
  own certificate is covered.
- **HTTP**: Traefik redirects HTTP to HTTPS for every app.
- **VPN**: a hardened, userspace Tailscale subnet router pod advertises only `10.0.0.100/32`. A
  tailnet policy limits the owner's devices to ports 80 and 443.
- **Where it lives**: three new infrastructure-tier charts, applied by `bootstrap/install.sh`
  after a diff review.
- **App changes**: HL Budget and nginx move to the new names by values changes merged through
  Argo CD, with the Auth0 URLs updated at the same moment.

## Technical Context

**Language/Version**: Helm 4.x charts and YAML (hl-helm), Bash (hl-helm scripts), Vault HCL policy.
Kubernetes 1.36 (k3s v1.36.5+k3s1).

**Primary Dependencies**:
- cert-manager v1.21.3 (upstream chart from `oci://quay.io/jetstack/charts`).
- Tailscale v1.104.4 (`tailscale/tailscale` image, pinned by digest).
- Traefik 3.7.13 (k3s-bundled chart 40.1.x), configured with `HelmChartConfig` and `TLSStore`.
- Vault Secrets Operator and the `hl-common` library chart (both existing).

**External services**:
- Cloudflare: registrar, DNS and an API token.
- Let's Encrypt: ACME.
- Tailscale: coordination server, policy and an OAuth client.
- Auth0: the existing tenant; only its URLs change.

**Storage**: none new. Certificate and Tailscale node state are Kubernetes Secrets written by the
controllers. The two credentials live in Vault at `kv/infra/cloudflare` and `kv/infra/tailscale`.

**Testing**:
- hl-helm `scripts/validate.sh`: helm lint, template, kubeconform, helm-unittest and the policy
  tests, with new unit tests for the three charts.
- A rehearsal on the k3d test cluster against the Let's Encrypt staging server.
- `install.sh --diff` review before each apply.
- Quickstart Part 3 live checks V1–V11.

**Target Platform**: single-node k3s homelab (`10.0.0.100`); clients are iOS, Android, macOS and
Windows browsers, with the Tailscale app when away.

**Project Type**: infrastructure and configuration for an existing web application (no service
code).

**Performance Goals**: no added latency at home (direct LAN path). Over the VPN, the page is
signed in within 30 s of connecting (SC-002).

**Constraints**:
- No inbound ports on the home router.
- Secrets only through Vault.
- Every workload is non-root with a read-only root filesystem and a memory limit.
- Infrastructure reaches the cluster only through `bootstrap/install.sh` from a committed tree
  (hl-helm Principle III).
- The certificate renews at least 2 weeks before expiry.

**Scale/Scope**:
- One owner, about 3 devices.
- Two apps today (hl-budget, nginx), more later.
- One certificate, one DNS record, one gateway.

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

HL Budget constitution v1.1.1:

| Principle | Check | Result |
|---|---|---|
| I. Spec-driven | This spec → plan → tasks before any change; an independently deployable increment | ✅ |
| II. Separate services, contract-first | No service, contract or OpenAPI change. Frontend and backend still share one host, `/` and `/api` | ✅ |
| III. Secure by default | Auth0 unchanged apart from the allowed URLs (FR-014). The two new credentials are stored only in Vault and synced by VSO, with nothing in git, values, images or logs (FR-017, V10). The network stays private: no public exposure, and the VPN reaches only ports 80 and 443 | ✅ |
| IV. Tests and gates | No service code changes, so the service gates are untouched. Chart changes pass hl-helm's gate, with unit tests for each new chart | ✅ |
| V. Homelab-ready operations | The Tailscale pod meets the container contract (UID 10001, non-root, read-only root filesystem, no capabilities, port 9002 > 1024, health probes). App changes flow hl-helm PR → Argo CD. Infrastructure flows through `install.sh`; there is no manual `kubectl apply` | ✅ |
| VI. Simplicity | Adds cert-manager and a Tailscale router, two new components | ⚠️ Justified in Complexity Tracking |
| VII. API design | No API change | ✅ n/a |

hl-helm constitution v3.0.0 (the repository this feature changes):

| Principle | Check | Result |
|---|---|---|
| I. Declarative IaC | Everything in the cluster comes from committed charts (including the Traefik `HelmChartConfig`). Outside the cluster: Cloudflare, Tailscale and Auth0 settings are documented as manual steps with exact values | ✅ |
| II. Chart per application | Each infrastructure chart has one purpose. App charts stay one per app | ✅ |
| III. Validate before apply | Infrastructure charts are explicitly applied, so the `--diff` review happens at apply. App charts are reviewed through their PR diff | ✅ |
| IV. Environment parity | Domain, email, ACME server, routes and tags are values; templates contain no environment branches | ✅ |
| V. Secrets never plaintext | Charts carry Vault paths only and fail when they are missing | ✅ |
| VI. Idempotent, reversible | Versioned charts, `helm rollback`, and the app change is one revertible PR. Tailscale state persists in a Secret | ✅ |
| VII. Simplicity, prefer k3s built-ins | cert-manager is an operator, while Traefik's built-in ACME is the k3s built-in | ⚠️ Justified below and in the chart README |
| Platform: ingress via controller with TLS; no NodePort/hostPort | Unchanged. The Tailscale pod has no Service and makes outbound connections only | ✅ |

**Gate result (pre-research)**: pass, with two justified complexity items.
**Re-check after Phase 1 design**: pass. The design adds no further components. Vault access is
narrowed with a dedicated `edge` role rather than widening `infra`.

## Project Structure

### Documentation (this feature)

```text
specs/005-home-and-remote-access/
├── spec.md
├── plan.md               # this file
├── research.md           # R-001 … R-014
├── data-model.md         # configuration objects, owners, states
├── quickstart.md         # manual steps M1–M11, rollout order, checks V1–V11
├── contracts/
│   ├── deployment.md     # hl-helm charts, values, Vault, DNS, Auth0
│   └── tailnet-policy.md # Tailscale policy file
├── checklists/requirements.md
└── tasks.md              # /speckit-tasks
```

### Source Code (repository root)

All changes are in `valyalkin/hl-helm` (branch `005-home-and-remote-access` there). The service
repositories are unchanged.

```text
hl-helm/
├── bootstrap/
│   ├── install-order.yaml          # + cert-manager, ingress-tls, tailscale (after redis)
│   ├── values/ingress-tls.yaml     # new: domain, acme.email, acme.server
│   └── vault-init.sh               # + policy and role "edge"
├── charts/
│   ├── infrastructure/
│   │   ├── cert-manager/           # new: wraps upstream v1.21.3 (CRDs on)
│   │   ├── ingress-tls/            # new: VSS, Issuer, Certificate, TLSStore, HelmChartConfig
│   │   └── tailscale/              # new: userspace subnet router Deployment, Role, VSS
│   └── apps/
│       ├── hl-budget/              # host budget.<domain>, APP_BASE_URL, drop own TLS secret
│       └── nginx/                  # host nginx.<domain>, guard annotation
├── tests/unit/                     # + tests for the three new charts
├── scripts/
│   ├── infra-status.sh             # + certificate Ready/expiry, tailscale, new namespaces
│   └── self-test.sh                # placeholder host updated
├── docs/
│   ├── onboarding.md               # app host naming; no TLS settings; no DNS step
│   ├── access.md                   # new names instead of hosts-file entries
│   └── remote-access.md            # new: Tailscale use, adding/removing devices
└── README.md                       # tier table, cluster baseline
```

In this repository: `specs/005-home-and-remote-access/` and, after deployment, a README Features
row (small separate PR).

**Structure Decision**: platform pieces are infrastructure-tier charts in hl-helm (research
R-009). The app change is a values change to the existing hl-budget and nginx charts. This keeps
the certificate and VPN independent of any one app (FR-016), and puts infrastructure behind the
apply-time diff review.

## Delivery slices

These map to the spec's stories and order `tasks.md`.

1. **Manual setup (owner)**: M1–M6 from the quickstart. This blocks everything else except
   writing charts.
2. **Foundation (P1 prerequisite)**:
   - the cert-manager and ingress-tls charts;
   - the Vault role `edge`;
   - install order and the status script;
   - a k3d rehearsal on staging, then apply on the homelab (staging → production).
3. **Story 1, home access (P1)**: the hl-budget values change and the Auth0 URLs (M9), merged
   together. Then the router check (M8), M10 and M11, and checks V1–V3 and V11.
4. **Story 2, remote access (P2)**: the tailscale chart, then apply and approve the route (M7).
   Checks V4, V5, V6 and V9.
5. **Story 3, future apps (P3)**: the nginx move and the onboarding docs. Check V7.
6. **Story 4, renewal (P3)**: the status-script output and a forced renewal. Check V8.
7. **Wrap-up**: V10 leak check, hl-helm docs and README, the spec README Features row, and the
   memory note.

## Risks

| Risk | Mitigation |
|---|---|
| Traefik ignores a `TLSStore` outside `kube-system` | Rehearse on k3d first (R-004). Fallback: `tlsStore.namespace: kube-system`, with the Certificate in the same namespace |
| Wrong redirect key for Traefik chart 40.x | Read the bundled chart's `values.yaml` before writing the `HelmChartConfig`, and test on k3d (R-005) |
| Tailscale userspace subnet routing fails as non-root with a read-only root filesystem | Rehearse on k3d with a throwaway auth key. Fallback: run as root in userspace mode (still no capabilities), recorded as a deviation in the chart README |
| Let's Encrypt rate limits while debugging | Staging server first; switch to production only after a staging certificate is issued |
| Router blocks private DNS answers | Check M8.2 with two fixes (rebind exception, or Tailscale on at home) |
| Sign-in breaks during the switch | The hl-budget PR merge and the Auth0 URL change happen together (M9). Rollback is reverting the PR and restoring the URLs |
| `HelmChartConfig` collides with a k3s-managed one | None exists today (checked). The chart owns the only one; documented in its README |

## Complexity Tracking

| Addition | Why needed | Simpler alternative rejected because |
|---|---|---|
| cert-manager (an operator with CRDs) | Wildcard certificate by DNS-01 with automatic renewal, visible as a `Certificate` with a `Ready` condition and expiry (FR-003–FR-005) | Traefik's built-in ACME keeps the certificate inside the k3s-managed Traefik (a file in its volume, token in `kube-system`). It is hard to inspect, not reusable outside Traefik, and its renewal state isn't visible to the status report. Self-signed or mkcert certificates need per-device setup (declined) |
| Tailscale subnet router (a new workload and an external service) | Remote access with no inbound ports, apps on every platform, device removal in one place (FR-009–FR-013) | Self-hosted WireGuard needs a forwarded port and per-device key handling. Cloudflare Tunnel is public exposure, not a VPN. Installing Tailscale on the node OS is outside hl-helm, so a rebuild would not restore it. The Tailscale operator adds an operator, CRDs and privileged proxies for a single route |
| Vault role `edge` | Least privilege for the two new credentials | Reusing `infra` would let each chart read every `kv/infra/*` secret, including Redis's |
