# Data Model: Home and Remote Access

**Feature**: [spec.md](spec.md) | **Date**: 2026-10-30

This feature stores no application data and needs no database migration. Its "entities" are
configuration objects across Cloudflare, the cluster, Vault, Tailscale and Auth0. This file
names each one, says where it lives and who changes it, and lists its validation rules and
states. The exact names and values are in [contracts/deployment.md](contracts/deployment.md).

## Domain

| Field | Value / rule |
|---|---|
| Name | `<domain>`; filled in after the purchase (manual step M1) |
| Registrar and DNS | Cloudflare, owner's account, two-factor authentication on |
| Auto-renew | MUST be on |
| Owned by | The owner (manual) |

States: `not bought` → `active` (Cloudflare shows *Active*) → `renewed` yearly. Losing `active`
breaks names and certificate renewal (spec Edge Cases).

## DNS record

| Field | Value / rule |
|---|---|
| Name | `*.<domain>` (wildcard). Apex `<domain>`: none until a personal page exists |
| Type / content | `A` → `10.0.0.100` (the node's private address) |
| Proxy | DNS only (Cloudflare cannot proxy a private address) |
| Owned by | The owner (manual step M3); changed again only if the node address changes |

A more specific record (for example a later public `budget.<domain>`) overrides the wildcard.

## App address

| Field | Value / rule |
|---|---|
| Host | `budget.<domain>` (HL Budget), `nginx.<domain>` (reference app) |
| Rule | One label under `<domain>`; unique across charts (hl-helm policy test `collisions.py`) |
| TLS | Uses Traefik's default certificate: `ingress.tls.secretName` unset |
| Lives in | Each app chart's `values.yaml` in hl-helm (for HL Budget, also `APP_BASE_URL`) |

## Cloudflare API token

| Field | Value / rule |
|---|---|
| Permission | `Zone → DNS → Edit`, zone resource limited to `<domain>` only |
| Stored | Vault `kv/infra/cloudflare`, key `api_token` (manual step M2) |
| Synced to | Secret `cloudflare-api-token` in namespace `ingress-tls`, by VSO |
| Read by | cert-manager's `Issuer` only |

Rotation: create a new token, write it to Vault, revoke the old one. VSO refreshes it within 1 h,
and nothing else changes (spec Edge Cases).

## Issuer

| Field | Value / rule |
|---|---|
| Kind / name | cert-manager `Issuer` `letsencrypt` in `ingress-tls` |
| Server | `staging` or `production`, from a chart value (`acme.server`) |
| Challenge | DNS-01, Cloudflare, token from the Secret above |
| Account email | The owner's email (chart value), for expiry notices from Let's Encrypt |

## Shared certificate

| Field | Value / rule |
|---|---|
| Kind / name | cert-manager `Certificate` `wildcard` in `ingress-tls` |
| DNS names | `<domain>`, `*.<domain>` |
| Secret | `wildcard-tls` (`kubernetes.io/tls`), written by cert-manager. Never in Vault or git |
| Renewal | Automatic, at two thirds of the lifetime (≥ 2 weeks before expiry; FR-004) |

States (the `Ready` condition): `Issuing` → `Ready=True` → near expiry, `Issuing` again →
`Ready=True`. On failure: `Ready=False` with a reason, and the old Secret keeps serving until it
expires. Shown by `scripts/infra-status.sh` (FR-005).

## Default TLS store

| Field | Value / rule |
|---|---|
| Kind / name | Traefik `TLSStore` `default` in `ingress-tls` (or `kube-system`, research R-004) |
| Default certificate | Secret `wildcard-tls` |
| Effect | Every ingress without its own `secretName` presents the wildcard certificate |

## Traefik configuration

| Field | Value / rule |
|---|---|
| Kind / name | k3s `HelmChartConfig` `traefik` in `kube-system` |
| Content | Permanent redirect from entry point `web` (80) to `websecure` (443) |
| Owned by | The `ingress-tls` chart; the only Traefik customisation in the cluster |

## Tailscale credential

| Field | Value / rule |
|---|---|
| Kind | Tailscale OAuth client with the `auth_keys` write scope, tag `tag:homelab` |
| Stored | Vault `kv/infra/tailscale`, key `client_secret` (manual step M5) |
| Synced to | Secret `tailscale-auth` in namespace `tailscale`, by VSO |
| Expiry | Does not expire; revoked in the Tailscale admin console |

## Homelab gateway (subnet router)

| Field | Value / rule |
|---|---|
| Tailnet name | `homelab`, tag `tag:homelab` |
| Advertised route | `10.0.0.100/32` only (FR-011) |
| State | Secret `tailscale-state` in namespace `tailscale` (written by the pod; holds its node key) |
| Key expiry | Disabled (tagged device), so no periodic sign-in (FR-013) |

States: `not registered` → `registered, route pending` → `route approved` (automatic through
`autoApprovers`; otherwise manual step M7) → `connected`. After a pod restart it comes back
`connected` from the saved state, without a new registration.

## VPN device

| Field | Value / rule |
|---|---|
| What | One of the owner's phones or laptops signed in to the tailnet |
| Allowed | `10.0.0.100:80,443` only (policy, [contracts/tailnet-policy.md](contracts/tailnet-policy.md)) |
| Removed by | The owner, in the Tailscale admin console *Machines* page (FR-012) |

## Auth0 application URLs

| Field | Before | After |
|---|---|---|
| Allowed Callback URLs | `https://hl-budget.home.example/auth/callback` | `https://budget.<domain>/auth/callback` |
| Allowed Logout URLs | `https://hl-budget.home.example` | `https://budget.<domain>` |
| Allowed Web Origins | `https://hl-budget.home.example` | `https://budget.<domain>` |

Changed by the owner (manual step M9) when the hl-budget values PR is merged. The API audience
`https://api.hl-budget.example` does not change.

## Vault access

| Role | Bound service accounts (namespace) | Policy reads |
|---|---|---|
| `edge` (new) | `ingress-tls` (`ingress-tls`), `tailscale` (`tailscale`) | `kv/data/infra/cloudflare`, `kv/data/infra/tailscale` |
| `apps`, `infra`, `provision` | unchanged | unchanged |

Removed: `kv/apps/hl-budget/tls` (empty; HL Budget no longer has its own certificate).
