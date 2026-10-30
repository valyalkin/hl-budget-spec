# Quickstart: Home and Remote Access

**Feature**: [spec.md](spec.md) | **Contracts**: [contracts/deployment.md](contracts/deployment.md),
[contracts/tailnet-policy.md](contracts/tailnet-policy.md)

**Domain**: `valyalkin.com` (so `<domain>` = `valyalkin.com`, HL Budget at
`https://budget.valyalkin.com`).

Part 1 lists the steps only the owner can do (accounts, payments, consoles). Part 2 is the order
of the whole rollout. Part 3 is the checks that prove the feature works. Replace `<domain>`
everywhere with the domain bought in M1.

Commands against the cluster use `KUBECONFIG=~/.kube/homelab.yaml` (through the owner's SSH
tunnel, see hl-helm `docs/access.md`). Vault commands use the hl-helm Vault access described
there too.

---

## Part 1: Manual steps (owner)

Total time: under 1 hour (SC-006).

### M1. Buy the domain (10 min)

1. Sign up at https://dash.cloudflare.com/sign-up and turn on two-factor authentication
   (*My Profile → Authentication*).
2. Add a payment method (*Manage Account → Billing*).
3. *Domain Registration → Register Domains*: search and buy a name (`.com`/`.net`/`.org`, about
   $10–12 a year, not "premium"). Turn **auto-renew on**.
4. Check *Websites*: the domain shows **Active**.

### M2. Cloudflare API token → Vault (5 min)

1. *My Profile → API Tokens → Create Token → Custom token*.
   - Permissions: **Zone → DNS → Edit**.
   - Zone resources: **Include → Specific zone → `<domain>`**.
   - No IP filter, no expiry (or a long one you put in a calendar).
2. Copy the token once and store it without echoing it:
   ```sh
   vault kv put kv/infra/cloudflare api_token=-   # paste, Enter, Ctrl-D
   ```

### M3. DNS record (2 min)

*Websites → `<domain>` → DNS → Records → Add record*: type **A**, name **`*`**, IPv4
**`10.0.0.100`**, proxy status **DNS only** (grey cloud). Check from any machine:

```sh
dig +short budget.<domain> @1.1.1.1      # expect 10.0.0.100
```

### M4. Tailscale account and policy (10 min)

1. Sign up at https://login.tailscale.com with the account you want to own the tailnet, and turn
   on two-factor authentication at that identity provider.
2. *Access controls*: paste the policy from
   [contracts/tailnet-policy.md](contracts/tailnet-policy.md) with your login filled in, and
   save. The built-in tests must pass.

### M5. Tailscale OAuth client → Vault (5 min)

1. *Settings → OAuth clients → Generate OAuth client*: scope **Keys → Auth Keys → Write**, tag
   **`tag:homelab`**.
2. Copy the client secret (`tskey-client-…`) once:
   ```sh
   vault kv put kv/infra/tailscale client_secret=-
   ```

### M6. Tailscale on your devices (5 min)

Install Tailscale on your phone (App Store / Play Store) and laptop
(https://tailscale.com/download), and sign in with the same account. On a laptop, make sure
subnet routes are used (macOS/Windows/iOS/Android use them by default; on Linux run
`tailscale set --accept-routes`).

### M7. Approve the homelab route (only if needed, 1 min)

After the `tailscale` release is applied (Part 2, step 6), *Machines → homelab* should show
the route `10.0.0.100/32` approved, through `autoApprovers`. If it shows "awaiting approval",
use *Edit route settings* and tick it.

### M8. Router (10 min)

1. **Reserve the node's address**: in the router's admin page (usually http://192.168.x.1 or
   http://10.0.0.1, printed on the router's label), find *DHCP reservation* / *Address
   reservation* / *Static lease* and reserve `10.0.0.100` for the homelab machine.
2. **Check DNS rebind protection**: from a laptop on home Wi-Fi with Tailscale **off**:
   ```sh
   dig +short budget.<domain>               # through the router
   dig +short budget.<domain> @1.1.1.1      # through the internet
   ```
   - Both print `10.0.0.100`: nothing to do.
   - Only the second prints it: the router blocks private answers. Look for *DNS rebind
     protection* in the router and add `<domain>` as an exception. If there is no such setting,
     keep Tailscale on at home on the affected devices (it resolves names itself).

### M9. Auth0 URLs (3 min, at the moment the hl-budget PR merges)

Auth0 dashboard → *Applications → HL Budget*:

- Allowed Callback URLs: `https://budget.<domain>/auth/callback`
- Allowed Logout URLs: `https://budget.<domain>`
- Allowed Web Origins: `https://budget.<domain>`

Remove the `hl-budget.home.example` entries, then *Save*.

### M10. Remove the old certificate path (1 min)

```sh
vault kv metadata delete kv/apps/hl-budget/tls
```

### M11. Remove old hosts entries (1 min per laptop)

```sh
sudo sed -i '' '/home\.example/d' /etc/hosts     # macOS
```

---

## Part 2: Rollout order

| # | Step | Who | Proof |
|---|---|---|---|
| 1 | M1–M5, M6 on the phone and laptop | Owner | Vault has both entries; DNS check in M3 passes |
| 2 | hl-helm PR: three new infrastructure charts, `vault-init.sh` role `edge`, `install-order.yaml`, `infra-status.sh`, docs. `scripts/validate.sh` green | Agent | CI green; merged |
| 3 | Re-run `bootstrap/vault-init.sh` (idempotent) to add policy and role `edge` | Owner/agent | `vault read auth/kubernetes/role/edge` |
| 4 | `bootstrap/install.sh --only cert-manager --diff`, review, then apply | Owner/agent | cert-manager pods Ready |
| 5 | `bootstrap/install.sh --only ingress-tls --diff` with `acme.server: staging`, review, apply; then switch to `production`, diff, apply | Owner/agent | `kubectl -n ingress-tls get certificate wildcard`: `READY True`; the staging issuer appears first, then Let's Encrypt |
| 6 | `bootstrap/install.sh --only tailscale --diff`, review, apply; M7 if needed | Owner/agent | *Machines* shows `homelab` connected with the route |
| 7 | hl-helm PR: hl-budget and nginx hosts, drop the hl-budget TLS secret; merge **together with M9** | Agent, owner for M9 | Argo CD `hl-budget` and `nginx` Synced/Healthy |
| 8 | M8, M10, M11 | Owner | Part 3 checks |

Rollback:
- Revert the step-7 PR and restore the Auth0 URLs.
- An infrastructure release is rolled back with `helm rollback <name> -n <ns>`. Removing
  `ingress-tls` returns Traefik to its self-signed default certificate.

---

## Part 3: Validation

Each check names the success criterion it proves.

### V1. Any device at home (SC-001, Story 1)

On a phone, a tablet and a laptop on home Wi-Fi, with Tailscale **off** and no hosts entries:

1. Open `https://budget.<domain>`. Expect the padlock and no warning.
2. Sign in. Expect a return to `https://budget.<domain>`, signed in.
3. Add a spending, then delete it.

Pass: 3 of 3 devices.

### V2. Certificate details (FR-002, FR-003)

```sh
echo | openssl s_client -connect 10.0.0.100:443 -servername budget.<domain> 2>/dev/null \
  | openssl x509 -noout -issuer -subject -ext subjectAltName -enddate
```

Expect: issuer Let's Encrypt (not "STAGING", not "TRAEFIK DEFAULT CERT"), SAN
`DNS:<domain>, DNS:*.<domain>`, and an end date in the future. Repeat with
`-servername nginx.<domain>`; expect the same certificate.

### V3. HTTP redirects to HTTPS (FR-006)

```sh
curl -sI http://budget.<domain>/ | sed -n '1p;/^[Ll]ocation/p'
```

Expect `301` or `308` and `Location: https://budget.<domain>/`.

### V4. Away from home (SC-002, Story 2)

On the phone, Wi-Fi **off** (mobile data):

1. Tailscale **on**. Open `https://budget.<domain>`, sign in, view this month. Time from
   connecting to the page loaded signed in: under 30 s.
2. Tailscale **off**. Reload. Expect it not to load (the private address is unreachable).

### V5. Only the homelab, only HTTP(S) (FR-011, Story 2 scenario 4)

On the laptop away from home (or a phone hotspot) with Tailscale on:

```sh
nc -vz -w 3 10.0.0.100 443   # succeeds
nc -vz -w 3 10.0.0.100 22    # fails (blocked by policy)
nc -vz -w 3 <router-ip> 80   # fails (other home devices are not routed through the VPN)
```

### V6. Nothing open to the internet (SC-003, FR-010)

From a network outside home (phone hotspot), with Tailscale **off**, find the home's public
address (`curl -s https://ifconfig.me` run **at home**), then:

```sh
nmap -Pn -p 1-1024,6443,8080,9002 <home-public-ip>
```

Expect no open ports. Also check the router's *Port forwarding* page is empty.

### V7. Second app with no certificate work (SC-004, Story 3)

After step 7, `https://nginx.<domain>` loads with no warning from a home phone and from the
VPN-connected phone. The nginx PR changed only the host (and guard/docs), with no DNS step and
no certificate settings.

### V8. Renewal works and is visible (SC-005, FR-004, FR-005, Story 4)

```sh
scripts/infra-status.sh             # in hl-helm: certificate READY True, expiry date shown
cmctl renew -n ingress-tls wildcard # or: kubectl cert-manager renew …
kubectl -n ingress-tls get certificate wildcard -w   # back to READY True with a new notAfter
```

V2 then shows the new end date, and the app never showed a warning. Failure drill (optional):
write a wrong token to `kv/infra/cloudflare`, force a renewal, and confirm `infra-status.sh`
reports the certificate as not ready while the site still serves the old certificate. Then
restore the token.

### V9. Gateway survives restarts (FR-013)

```sh
kubectl -n tailscale rollout restart deploy/tailscale && kubectl -n tailscale rollout status deploy/tailscale
```

The *Machines* page still shows a single `homelab` (no `homelab-1` duplicate), connected. V4
step 1 passes again.

### V10. No credentials leaked (SC-007, FR-017)

```sh
# in hl-helm and hl-budget-spec
git grep -nE 'tskey-|cf[_-]?api|api_token *[:=] *[A-Za-z0-9_-]{20,}' -- . ':!*.md' || echo clean
kubectl -n tailscale logs deploy/tailscale | grep -c tskey-   # expect 0
```

### V11. Old address retired (FR-014, FR-015)

`https://hl-budget.home.example` (using `curl -k --resolve hl-budget.home.example:443:10.0.0.100`)
returns 404 from Traefik. Auth0 *Allowed Callback URLs* no longer lists it.
