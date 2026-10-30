# Contract: Platform and deployment changes in hl-helm

**Feature**: [../spec.md](../spec.md) | **Research**: [../research.md](../research.md)

This is the interface between this feature and `valyalkin/hl-helm`. It lists what the new charts
accept and render, what changes in existing charts and scripts, and the external settings. No
service API changes (the HL Budget OpenAPI contract is untouched).

## 1. Install order (`bootstrap/install-order.yaml`)

The three new releases go after `redis` and before Argo CD:

```yaml
  - name: cert-manager
    chart: charts/infrastructure/cert-manager
    namespace: cert-manager
  - name: ingress-tls
    chart: charts/infrastructure/ingress-tls
    namespace: ingress-tls
    values: bootstrap/values/ingress-tls.yaml
  - name: tailscale
    chart: charts/infrastructure/tailscale
    namespace: tailscale
```

They are applied with `bootstrap/install.sh --only <name>`, after reviewing `--diff`, from a
committed tree.

## 2. `charts/infrastructure/cert-manager`

- Wraps upstream `cert-manager` **v1.21.3** (`oci://quay.io/jetstack/charts`) under the key
  `cert-manager`, as `cloudnative-pg` wraps its upstream chart.
- Values: `crds.enabled: true`; memory limits on controller, webhook and cainjector;
  `dns01RecursiveNameservers: "1.1.1.1:53,8.8.8.8:53"` and
  `dns01RecursiveNameserversOnly: true`, so the DNS-01 self-check asks public resolvers rather
  than the cluster DNS.
- No secret references.

## 3. `charts/infrastructure/ingress-tls`

Depends on `hl-common` (for `hl-common.namespace` with `vaultAuth: {role: edge,
createNamespace: false}` and `hl-common.vaultStaticSecret`).

**Values** (all required unless a default is shown; `bootstrap/values/ingress-tls.yaml` holds the
real domain and email):

| Key | Example | Rule |
|---|---|---|
| `domain` | `example.com` | Required; no scheme, no wildcard. Rendering fails if unset |
| `acme.email` | `owner@example.com` | Required |
| `acme.server` | `staging` | `staging` (default, safe) or `production` |
| `cloudflare.vaultPath` | `infra/cloudflare` | Default shown; key `api_token` |
| `redirectHttpToHttps` | `true` | Default `true`; renders the `HelmChartConfig` |
| `tlsStore.namespace` | `ingress-tls` | Default the release namespace; `kube-system` if R-004 requires it |

**Renders**: ServiceAccount + `VaultAuth` (role `edge`); `VaultStaticSecret` →
Secret `cloudflare-api-token`; `Issuer` `letsencrypt`; `Certificate` `wildcard`
(`<domain>`, `*.<domain>` → Secret `wildcard-tls`); `TLSStore` `default`; `HelmChartConfig`
`traefik` in `kube-system` with the web → websecure permanent redirect.

**Unit tests** (helm-unittest): fails without `domain`/`acme.email`; DNS names are exactly the
apex and the wildcard; `staging`/`production` map to the two Let's Encrypt URLs; no secret
material in any rendered object; redirect toggled by `redirectHttpToHttps`.

## 4. `charts/infrastructure/tailscale`

Depends on `hl-common` (namespace binding with role `edge`, secrets, labels, probes, security
context).

**Values**:

| Key | Default | Rule |
|---|---|---|
| `image` | `tailscale/tailscale` `v1.104.4` + digest | Pinned; never `latest` |
| `routes` | — | Required; `["10.0.0.100/32"]` |
| `hostname` | `homelab` | Tailnet machine name |
| `tags` | `["tag:homelab"]` | Required non-empty (an OAuth key needs a tag) |
| `auth.vaultPath` | `infra/tailscale` | Key `client_secret` |
| `resources.limits.memory` | `128Mi` | Required by the platform |

**Renders**: ServiceAccount + `VaultAuth`; `VaultStaticSecret` → Secret `tailscale-auth`; a
Role/RoleBinding letting the service account `get`/`update`/`patch` only the Secret
`tailscale-state`; one-replica Deployment (strategy `Recreate`) with:

- env: `TS_USERSPACE=true`, `TS_ROUTES`, `TS_HOSTNAME`, `TS_KUBE_SECRET=tailscale-state`,
  `TS_EXTRA_ARGS=--advertise-tags=<tags>`,
  `TS_AUTHKEY=<client_secret>?ephemeral=false&preauthorized=true`, `TS_ENABLE_HEALTH_CHECK=true`,
  `TS_LOCAL_ADDR_PORT=[::]:9002`, socket and tmp under `/tmp`;
- the security context: UID/GID 10001, non-root, read-only root filesystem, all capabilities
  dropped, no privilege escalation;
- liveness and readiness probes on `/healthz` (port 9002);
- no Service or Ingress (it only makes outbound connections).

**Unit tests**: fails without `routes` or `tags`; the security context is set; the Role names
exactly one Secret; no `NET_ADMIN`, `privileged` or `hostNetwork`.

## 5. Vault (`bootstrap/vault-init.sh`)

Add, idempotently:

```hcl
# policy "edge"
path "kv/data/infra/cloudflare" { capabilities = ["read"] }
path "kv/data/infra/tailscale"  { capabilities = ["read"] }
```

```sh
vault write auth/kubernetes/role/edge \
  bound_service_account_names=ingress-tls,tailscale \
  bound_service_account_namespaces=ingress-tls,tailscale \
  audience=vault token_policies=edge ttl=1h
```

The owner writes the values (manual steps M2, M5):

```sh
vault kv put kv/infra/cloudflare api_token=-     # read from stdin, never on the command line
vault kv put kv/infra/tailscale client_secret=-
```

and deletes `kv/apps/hl-budget/tls` (step M10).

## 6. `charts/apps/hl-budget` (Argo CD; a merge deploys it)

```diff
 backend:
   ingress:
-    host: hl-budget.home.example
+    host: budget.<domain>
-    tls: {secretName: hl-budget-frontend-tls}
 frontend:
   workload:
     env:
-      - {name: APP_BASE_URL, value: "https://hl-budget.home.example"}
+      - {name: APP_BASE_URL, value: "https://budget.<domain>"}
   ingress:
-    host: hl-budget.home.example
+    host: budget.<domain>
-    tls: {secretName: hl-budget-frontend-tls}
   secrets:
-    tls: {vaultPath: apps/hl-budget/tls, type: kubernetes.io/tls, keys: {}, rolloutRestart: false}
```

Chart `version` bumped and README regenerated (helm-docs). The `APP_BASE_URL` change restarts the
frontend, which is expected.

## 7. `charts/apps/nginx` and repository docs

- `web.ingress.host: nginx.<domain>`. Update the reference-chart guard
  (`hl-common/placeholders`) in `charts/apps/nginx/Chart.yaml` and
  `charts/apps/hl-budget/Chart.yaml`, `scripts/self-test.sh`, and `docs/access.md`.
- `docs/onboarding.md`: app hosts are `<name>.<domain>`, `tls.secretName` stays unset, and no DNS
  step is needed (wildcard record).
- `docs/remote-access.md` (new): Tailscale setup, policy, adding and removing a device.
- `README.md`: the three new infrastructure releases and the tier table.
- `scripts/infra-status.sh`: certificate `Ready` and `notAfter`, Tailscale pod state, the new
  namespaces in the unhealthy-pod check.

## 8. Cloudflare

| Record | Type | Content | Proxy |
|---|---|---|---|
| `*.<domain>` | A | `10.0.0.100` | DNS only |

## 9. Auth0 (HL Budget application)

| Setting | Value |
|---|---|
| Allowed Callback URLs | `https://budget.<domain>/auth/callback` |
| Allowed Logout URLs | `https://budget.<domain>` |
| Allowed Web Origins | `https://budget.<domain>` |

The old `hl-budget.home.example` entries are removed at the same time.

## 10. Tailnet

See [tailnet-policy.md](tailnet-policy.md).

## Compatibility

- No API change, so nothing is breaking under constitution VII.
- The old address stops working when the hl-budget PR merges. Rollback is reverting that PR and
  restoring the Auth0 URLs.
- Infrastructure changes affect every app: during the first apply the default certificate changes
  for `nginx` too, which is intended.
