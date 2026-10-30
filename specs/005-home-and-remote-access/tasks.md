---

description: "Task list for 005-home-and-remote-access"
---

# Tasks: Home and Remote Access

**Input**: Design documents from `/specs/005-home-and-remote-access/`

**Prerequisites**: plan.md, spec.md, research.md, data-model.md, contracts/deployment.md,
contracts/tailnet-policy.md, quickstart.md

**Tests**: REQUIRED for chart code. hl-helm Principle III and research R-014 require
helm-unittest tests for each new chart, the `scripts/validate.sh` gate, and a `--diff` review
before each infrastructure apply. Write each chart's tests first and see them fail. The live
checks V1–V11 in quickstart.md Part 3 are the acceptance tests for the stories.

**Organization**: Tasks are grouped by user story (spec.md US1–US4). Tasks marked
**MANUAL (owner)** need the owner's accounts, payment or consoles; an agent cannot do them. When
an agent reaches one, it stops and asks the owner, then records the outcome (never a secret
value) in the task.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Can run in parallel (different files, no dependency on an unfinished task).
- **[Story]**: The user story the task belongs to (US1 … US4).
- `· skill: <name>` at the end of a task: load that skill before starting. Where a skill and a
  constitution disagree, the constitution wins (the hl-helm constitution for chart work).

## Path Conventions

All paths are relative to the spec repo root `/Users/valyalkin/Documents/Projects/hl-budget-spec/`.

- `hl-helm/`: a git worktree of `valyalkin/hl-helm` on branch `005-home-and-remote-access`
  (T001). Every chart, script and doc change happens there.
- `specs/005-home-and-remote-access/`: this feature's documents, written below as `feat/`.
- Cluster commands use `KUBECONFIG=~/.kube/homelab.yaml` (the owner's SSH tunnel must be up).
  The local test cluster is the default context `k3d-homelab-test`. Never switch the default
  context.
- `<domain>` means the domain bought in T002, and `<email>` the owner's email for Let's Encrypt
  notices.
- No task changes `hl-budget-backend` or `hl-budget-frontend`.

---

## Phase 1: Setup

**Purpose**: The working copy, the owner's accounts and credentials, and the one fact to confirm
before writing charts.

- [X] T001 Create the hl-helm worktree and keep it out of this repo's git:
  `git -C ../hl-helm fetch origin && git -C ../hl-helm worktree add "$PWD/hl-helm" -b 005-home-and-remote-access origin/main`.
  Add `/hl-helm/` to `.gitignore` and `!/hl-helm/` to `.ignore`, with the existing comments'
  wording extended to "the service and platform repositories".
- [X] T002 [P] **MANUAL (owner)**: quickstart **M1**. Create the Cloudflare account with
  two-factor authentication, buy `<domain>` with **auto-renew on**, and confirm *Websites*
  shows **Active**. Record the domain name in this task and at the top of `feat/quickstart.md`
  ("Domain: …").
  *Done 2026-10-30: `valyalkin.com`, registered with Cloudflare 2026-10-30 (expires 2027-10-30,
  auto-renew), nameservers `abby`/`buck.ns.cloudflare.com`.*
- [X] T003 [P] **MANUAL (owner)**: quickstart **M4**. Create the Tailscale account (two-factor
  authentication at the identity provider), then paste the policy from
  `feat/contracts/tailnet-policy.md` with `<owner-login>` filled in, and save. The policy's
  built-in `tests` must pass.
  *Done 2026-10-30: tailnet created with Google login `valyalkin@gmail.com`; policy saved (tests
  passed), with `tagOwners` using `autogroup:admin` (contract updated).*
- [X] T004 **MANUAL (owner)** (after T002): quickstart **M2**. Create a Cloudflare custom API
  token with permission **Zone → DNS → Edit**, limited to "Specific zone → `<domain>`". Store it
  with `vault kv put kv/infra/cloudflare api_token=-` (stdin, never on the command line).
  Confirm with `vault kv metadata get kv/infra/cloudflare`.
  *Done 2026-10-30: custom token "homelab cert-manager" (Zone · DNS · Edit, valyalkin.com only, no
  expiry), stored by the owner through the Vault UI at `kv/infra/cloudflare` key `api_token`.
  Read-back is confirmed when VSO syncs it (T022).*
  *Incident 2026-10-30: the `kv/` secrets engine was disabled by mistake in the Vault UI, which
  erased every kv secret. Restored the same day by the owner: kv-v2 re-enabled at `kv/` and the
  four app secrets re-written from the VSO-synced Kubernetes Secrets (a private backup is in
  `~/.hl-helm/vault-kv-recovery-2026-10-30.json`). Values verified identical by hash, VSO synced
  all paths, apps healthy. `infra/cloudflare` re-saved. Follow-up idea (not this feature):
  scheduled Vault snapshots.*
- [X] T005 [P] **MANUAL (owner)** (after T002): quickstart **M3**. Add the record `*` **A**
  `10.0.0.100`, proxy **DNS only**. Check: `dig +short budget.<domain> @1.1.1.1` prints
  `10.0.0.100`.
  *Done 2026-10-30: `*.valyalkin.com A 10.0.0.100` (DNS only); `budget.` and `nginx.valyalkin.com`
  resolve to `10.0.0.100` through 1.1.1.1.*
- [X] T006 **MANUAL (owner)** (after T003): quickstart **M5**. Create an OAuth client with scope
  **Keys → Auth Keys → Write** and tag **`tag:homelab`**. Store the secret with
  `vault kv put kv/infra/tailscale client_secret=-`.
  *Done 2026-10-30: OAuth client "homelab gateway",
  scope Auth Keys: Write, tag `tag:homelab`; stored at `kv/infra/tailscale` key `client_secret`
  (`tskey-client-…`, checked by prefix only).*
- [X] T007 [P] **MANUAL (owner)** (after T003): quickstart **M6**. Install Tailscale on the
  phone and laptop and sign in. Confirm both appear under *Machines*.
  *Done 2026-10-30 for the Mac (connected). The phone is still to do before quickstart V4 (T039).*
- [X] T008 [P] Confirm the HTTP→HTTPS redirect key for the k3s-bundled Traefik chart
  (research R-005):
  `KUBECONFIG=~/.kube/homelab.yaml kubectl get --raw /static/charts/traefik-40.1.5+up40.1.0.tgz > <scratchpad>/traefik.tgz`,
  then `helm show values <scratchpad>/traefik.tgz | grep -n -A6 redirections`. Record the exact key path
  (`ports.web.http.redirections.entryPoint` or `ports.web.redirections.entryPoint`) and its
  fields under R-005 in `feat/research.md`.
  *Done 2026-10-30: `ports.web.http.redirections.entryPoint` (recorded in R-005).*

**Checkpoint**: domain active, both credentials in Vault, DNS answers `10.0.0.100`, Tailscale on
the owner's devices, redirect key known.

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: The shared trusted certificate, Traefik's default certificate and the HTTP
redirect, live on the homelab. US1, US3 and US4 need this; US2 needs it too, because the VPN
serves the same certificate.

**⚠️ CRITICAL**: No user story work can begin until this phase is complete.

### Tests first

- [X] T009 Extend `hl-helm/scripts/validate.sh` so the "unit tests" step also runs
  `helm unittest -f 'tests/*_test.yaml' <chart>` for every `charts/*/*/` that has a `tests/`
  directory, after `helm dependency build`. Update `hl-helm/scripts/self-test.sh` so one injected
  violation proves a failing chart test fails the gate. Run
  `hl-helm/scripts/validate.sh --self-test`: it passes.
- [X] T010 [P] Write `hl-helm/charts/infrastructure/ingress-tls/tests/ingress_tls_test.yaml`
  (helm-unittest), covering contract §3:
  - rendering fails with a clear message when `domain` or `acme.email` is unset;
  - the `Certificate` `wildcard` has `dnsNames` exactly `[<domain>, "*.<domain>"]`,
    `secretName: wildcard-tls` and `issuerRef` `Issuer/letsencrypt`;
  - `acme.server: staging` renders `https://acme-staging-v02.api.letsencrypt.org/directory`,
    `production` renders `https://acme-v02.api.letsencrypt.org/directory`, and any other value
    fails (schema);
  - the `Issuer` solver is `dns01.cloudflare.apiTokenSecretRef {name: cloudflare-api-token, key: api_token}`;
  - the `VaultStaticSecret` uses `vaultAuthRef` with role `edge`, path `infra/cloudflare` and
    destination `cloudflare-api-token`;
  - the `TLSStore` is named `default` with `defaultCertificate.secretName: wildcard-tls`, in
    `tlsStore.namespace` (default: the release namespace);
  - the `HelmChartConfig` `traefik` is in `kube-system` when `redirectHttpToHttps: true` and
    absent when `false`, using the key from T008;
  - no rendered object contains secret material.

  Run it; it fails (the chart does not exist yet).

### cert-manager and ingress-tls charts

- [X] T011 [P] Add the Vault policy and role `edge` to `hl-helm/bootstrap/vault-init.sh`,
  idempotently and next to the existing roles. The text is in contract §5: policy reads
  `kv/data/infra/cloudflare` and `kv/data/infra/tailscale` only; role bound to service accounts
  `ingress-tls,tailscale` in namespaces `ingress-tls,tailscale`, `audience=vault`, `ttl=1h`.
  Update the final echo line to list role `edge`.
- [X] T012 [P] Create `hl-helm/charts/infrastructure/cert-manager/`, copying the structure of
  `charts/infrastructure/cloudnative-pg/`:
  - `Chart.yaml`: `version: 0.1.0`, `appVersion: "v1.21.3"`, dependency `cert-manager`
    `v1.21.3` from `oci://quay.io/jetstack/charts`.
  - `values.yaml`, under the key `cert-manager`: `crds.enabled: true`; requests and memory
    limits for controller, webhook and cainjector (`64Mi`/`128Mi` each to start);
    `extraArgs: ["--dns01-recursive-nameservers=1.1.1.1:53,8.8.8.8:53", "--dns01-recursive-nameservers-only"]`.
  - `values.schema.json` and `ci/default-values.yaml`.
  - `README.md.gotmpl`, which must document the hl-helm Principle VII deviation ("an operator
    instead of Traefik's built-in ACME") with the reason from plan.md Complexity Tracking, and
    the expiry condition "revisit if Traefik-native certificates become inspectable and
    shareable".

  Run `helm dependency update`.
- [X] T013 Create `hl-helm/charts/infrastructure/ingress-tls/` per contract §3, depending on
  `hl-common` (`file://../../library/hl-common`):
  - `Chart.yaml`: `version: 0.1.0`.
  - `values.yaml`, with every key documented for helm-docs: `domain` (empty),
    `acme.email` (empty), `acme.server: staging`,
    `cloudflare.vaultPath: infra/cloudflare`, `redirectHttpToHttps: true`,
    `tlsStore.namespace: ""`, and `vaultAuth: {role: edge, createNamespace: false}`.
  - `values.schema.json`: `acme.server` enum `[staging, production]`; `domain` a hostname
    pattern with no scheme or `*`.
  - templates:
    - `namespace.yaml`: `{{ include "hl-common.namespace" . }}` (ServiceAccount
      `ingress-tls` and VaultAuth).
    - `cloudflare-secret.yaml`: a `VaultStaticSecret` → Secret `cloudflare-api-token`, with
      hl-common labels and no rollout target.
    - `issuer.yaml`: `Issuer` `letsencrypt`, ACME, with the email and server from values and
      the Cloudflare DNS-01 solver.
    - `certificate.yaml`: `Certificate` `wildcard`.
    - `tlsstore.yaml`: `TLSStore` `default` (`traefik.io/v1alpha1`).
    - `helmchartconfig.yaml`: `HelmChartConfig` `traefik` in `kube-system`
      (`helm.cattle.io/v1`), with the redirect key from T008 (`to: websecure`,
      `scheme: https`, `permanent: true`).
  - `ci/default-values.yaml` with `domain: example.com` and `acme.email: owner@example.com`.
  - `README.md.gotmpl`: what each object does, rotating the Cloudflare token, staging →
    production, and that this chart owns the cluster's only Traefik `HelmChartConfig`.

  T010 now passes.
- [X] T014 Create `hl-helm/bootstrap/values/ingress-tls.yaml` with `domain: <domain>` (from
  T002), `acme.email: <email>` and `acme.server: staging`.
- [X] T015 Add the `cert-manager` and `ingress-tls` entries to
  `hl-helm/bootstrap/install-order.yaml`, after `redis`, exactly as in contract §1 (tailscale
  comes in T037).
- [X] T016 [P] Extend `hl-helm/scripts/infra-status.sh`:
  - a "Certificates" section printing, for each `Certificate` in namespace `ingress-tls`, its
    name, `READY` (the `Ready` condition), `status.notAfter` and days left. Days left under 14
    is flagged `RENEWAL OVERDUE`; `READY False` prints the condition message.
  - add `cert-manager ingress-tls tailscale` to the unhealthy-pod namespaces.
- [X] T017 Run `hl-helm/scripts/validate.sh` (and `--self-test`) until `GATE PASS`. Commit the
  output of `helm-docs`.

### Rehearse, then apply

- [X] T018 Rehearse on the local test cluster `k3d-homelab-test` (its built-in Traefik), with the
  Let's Encrypt **staging** server:
  1. Install `charts/infrastructure/cert-manager`.
  2. Create the Secret `cloudflare-api-token` in `ingress-tls` by hand from the owner's token.
     This is a debug-only exception on a disposable cluster, deleted at the end of this task.
  3. Install `ingress-tls` with `redirectHttpToHttps: true`.

  Verify, then record the results under R-004 and R-005 in `feat/research.md`:
  - the `Certificate` becomes Ready;
  - `curl -vk --resolve nginx.<domain>:443:<k3d-lb-ip> https://nginx.<domain>/` presents the
    staging wildcard certificate, which proves the `TLSStore` outside `kube-system` works;
  - `curl -sI http://…` returns a permanent redirect to https.

  If the TLSStore is ignored, set `tlsStore.namespace: kube-system` (and move the Certificate to
  match) and repeat. Uninstall and delete the namespaces afterwards.
  *Done 2026-10-30: staging wildcard issued; the TLSStore outside `kube-system` works; 308 redirect.
  Results are in R-004 and R-005. Found and fixed during the rehearsal: the hand-made test secret
  had a trailing newline (`jq -r`; use `jq -j`). The Vault value itself is clean. k3d cleaned up
  (releases, namespaces and the token copy removed; cert-manager CRDs remain, as `crds.keep`
  intends).*
- [X] T019 Open the hl-helm PR "005: cert-manager, ingress-tls, Vault role edge" from `hl-helm/`
  (Conventional Commit messages). The description includes the rendered templates for the two
  new charts and the T018 results. CI green, then merge.
  *Done 2026-10-30: hl-helm#21 merged (CI gate and rendered-diff green).*
- [X] T020 On the homelab, from the merged and pulled `main`, re-run
  `hl-helm/bootstrap/vault-init.sh` (idempotent). Check `vault read auth/kubernetes/role/edge`.
  *Done 2026-10-30: role `edge` is live (service accounts ingress-tls/tailscale; the policy reads
  the two paths only). Existing kv data untouched.*
- [X] T021 `bootstrap/install.sh --context homelab --only cert-manager --diff`, have the owner
  review it, then apply without `--diff`. Check that the cert-manager pods are Ready.
  *Done 2026-10-30: diff reviewed by the owner (additions only) and applied; 3 pods Running.
  `install.sh --diff` needed `--allow-unreleased` for a new release (fixed in hl-helm#22).*
- [X] T022 Same for `--only ingress-tls` (staging). Check:
  - `kubectl -n ingress-tls get certificate wildcard` shows `READY True`;
  - the VSO-synced Secret exists;
  - `openssl s_client -connect 10.0.0.100:443 -servername budget.<domain>` shows a "(STAGING)"
    Let's Encrypt issuer;
  - `curl -sI http://10.0.0.100/` returns a permanent redirect.
  *Done 2026-10-30: owner approved diff (7 additions); certificate READY at once, VSS synced; openssl shows `(STAGING)` issuer, SAN `*.valyalkin.com, valyalkin.com`; `http://` → 308 to https; old hl-budget host still 200; infra-status: `wildcard True`, 63 days left.*
- [X] T023 Set `acme.server: production` in `hl-helm/bootstrap/values/ingress-tls.yaml` (small
  PR, merge). Run `--only ingress-tls --diff`, review, apply. If the Certificate does not reissue
  by itself, delete Secret `wildcard-tls` to force it. Quickstart **V2** passes: Let's Encrypt
  production issuer, SAN `<domain>, *.<domain>`.
  *Done 2026-10-30: hl-helm#22 merged; owner approved the one-line Issuer diff. The Issuer change did
  not reissue on its own, so `wildcard-tls` was deleted, and the production certificate was READY
  about 2.5 minutes later. V2: issuer `Let's Encrypt YE1`, SAN `*.valyalkin.com, valyalkin.com`,
  valid until 2027-01-28. `curl https://budget.valyalkin.com/` from the Mac verifies TLS with no
  flags.*

**Checkpoint**: every app on the homelab is served with the trusted wildcard certificate, and
HTTP redirects to HTTPS.

---

## Phase 3: User Story 1 — Use HL Budget from any device at home (Priority: P1) 🎯 MVP

**Goal**: `https://budget.<domain>` works on any home device with no warning and no setup.

**Independent Test**: quickstart **V1**: a phone, a tablet and a laptop on home Wi-Fi with
Tailscale off and no hosts entries load the app, sign in and add a spending.

- [X] T024 [US1] In `hl-helm/charts/apps/hl-budget/values.yaml`, apply contract §6 exactly:
  - both `ingress.host` values become `budget.<domain>`;
  - `APP_BASE_URL` becomes `https://budget.<domain>`;
  - remove both `ingress.tls` blocks and `frontend.secrets.tls`;
  - update the header comment ("one private host").

  Bump `version` in `hl-helm/charts/apps/hl-budget/Chart.yaml` (patch), regenerate the README
  with `helm-docs`, and run `scripts/validate.sh` until `GATE PASS`.
  · skill: homelab:hl-service-config
  *Done 2026-10-30: branch `config/hl-budget-host`, chart 0.2.10 → 0.2.11. The rendered diff is
  exactly contract §6 (hosts, APP_BASE_URL, TLS VSS removed). GATE PASS.*
- [X] T025 [US1] Open the hl-helm PR "hl-budget: move to budget.<domain>" with the rendered
  ingress and env diff, and wait for CI to pass. **Do not merge yet.**
  *Done 2026-10-30: hl-helm#23, CI green, not merged.*
- [X] T026 [US1] **MANUAL (owner)**: quickstart **M9**, done together with merging T025. In the
  Auth0 dashboard, set Allowed Callback URLs to `https://budget.<domain>/auth/callback`, Allowed
  Logout URLs and Allowed Web Origins to `https://budget.<domain>`, and remove the
  `hl-budget.home.example` entries. Then merge the T025 PR.
  *In progress 2026-10-30: the owner added the new URLs alongside the old ones (no sign-in gap)
  and hl-helm#23 was merged. After signing in on the new address, the owner removed the old
  entries.*
- [X] T027 [US1] Wait for Argo CD to sync (about 6–10 min) and confirm `hl-budget` is
  Synced/Healthy, with the frontend restarted and the ingresses showing host `budget.<domain>`
  with no `secretName`. · skill: homelab:hl-deploy-status
  *Done 2026-10-30: Synced/Healthy about 30 s after a manual Argo CD refresh. Both ingresses have
  host `budget.valyalkin.com` with no secret; the TLS VSS is gone; `APP_BASE_URL` updated. From
  the Mac with no flags: `/` 200, `/auth/login` 307 to Auth0 with
  `redirect_uri=https://budget.valyalkin.com/auth/callback`, `/api/v1/me` 401, `http://` 308. The
  old host returns 404.*
- [ ] T028 [US1] **MANUAL (owner)**: quickstart **M8**. Reserve `10.0.0.100` for the homelab
  machine in the router. Run the two `dig` commands from a laptop on home Wi-Fi with Tailscale
  off. If only the `@1.1.1.1` one answers, add a DNS rebind exception for `<domain>`; if the
  router has no such setting, note "keep Tailscale on at home" in this task. Record the result.
- [ ] T029 [P] [US1] **MANUAL (owner)**: quickstart **M10** and **M11**. Run
  `vault kv metadata delete kv/apps/hl-budget/tls`, then remove the `home.example` lines from
  `/etc/hosts` on each laptop.
- [ ] T030 [US1] Run quickstart **V1** (owner, three devices), **V2**, **V3** and **V11**, and
  record pass/fail per check in this task. Any failure blocks the phase.

**Checkpoint**: MVP. HL Budget works on every home device at its new address.

---

## Phase 4: User Story 2 — Use HL Budget away from home over a VPN (Priority: P2)

**Goal**: The same address works away from home with Tailscale on; nothing is open to the
internet; the VPN reaches only the homelab's ports 80 and 443.

**Independent Test**: quickstart **V4**, **V5**, **V6** and **V9**.

- [X] T031 [P] [US2] Write `hl-helm/charts/infrastructure/tailscale/tests/tailscale_test.yaml`
  (helm-unittest), covering contract §4:
  - rendering fails without `routes` or with empty `tags`;
  - the Deployment has 1 replica and strategy `Recreate`;
  - the container env has `TS_USERSPACE=true`, `TS_ROUTES=10.0.0.100/32`,
    `TS_HOSTNAME=homelab`, `TS_KUBE_SECRET=tailscale-state`,
    `TS_EXTRA_ARGS=--advertise-tags=tag:homelab`, `TS_ENABLE_HEALTH_CHECK=true` and
    `TS_LOCAL_ADDR_PORT=[::]:9002`;
  - `TS_AUTHKEY` comes from Secret `tailscale-auth` (the `?ephemeral=false&preauthorized=true`
    suffix is added at runtime, never stored in values);
  - the security context has `runAsUser: 10001`, `runAsNonRoot: true`,
    `readOnlyRootFilesystem: true`, `capabilities.drop: [ALL]` and
    `allowPrivilegeEscalation: false`;
  - there is no `NET_ADMIN`, `privileged` or `hostNetwork`;
  - liveness and readiness probes hit `/healthz` on port 9002;
  - the Role grants `get`/`update`/`patch` on `secrets` with `resourceNames: [tailscale-state]`
    only;
  - the image is pinned by digest;
  - a memory limit is set.

  Run it; it fails.
- [X] T032 [US2] Rehearse the gateway on `k3d-homelab-test` before writing the chart (research
  R-006):
  - Run a throwaway pod with `tailscale/tailscale:v1.104.4`, UID 10001, a read-only root
    filesystem, `/tmp` as emptyDir, and `TS_USERSPACE=true`, `TS_SOCKET=/tmp/tailscaled.sock`,
    `TS_STATE_DIR` unset, `TS_KUBE_SECRET`, `TS_ROUTES=<k3d node ip>/32`, `TS_HOSTNAME=homelab-test`.
  - Authenticate with an **ephemeral** pre-approved auth key the owner creates for this test.
  - Confirm from the phone (Tailscale on, Wi-Fi off) that `https://<k3d node ip>` answers
    through the route, and that `/healthz` on 9002 responds.

  Record the working env set and any deviation under R-006 in `feat/research.md`. If non-root
  is impossible, record the fallback "root in userspace mode, no capabilities" for the chart
  README. The owner removes the test machine and revokes the key afterwards.
  *Done 2026-10-30, with the chart itself and the real OAuth client in ephemeral mode (no extra
  key). Results and contract changes are in R-006. The end-to-end path can't be tested on k3d
  (Docker Desktop has no LAN access), so it moves to V4. k3d cleaned up; the ephemeral
  `homelab-test` machine expires on its own.*
- [X] T033 [US2] Create `hl-helm/charts/infrastructure/tailscale/` per contract §4, depending on
  `hl-common`:
  - `Chart.yaml`: `version: 0.1.0`, `appVersion: "v1.104.4"`.
  - `values.yaml`: `image` (repository `tailscale/tailscale`, tag `v1.104.4`, digest resolved
    with `crane digest` or `docker buildx imagetools inspect`), `routes: []`,
    `hostname: homelab`, `tags: [tag:homelab]`, `auth.vaultPath: infra/tailscale`,
    `resources.limits.memory: 128Mi` with requests, and
    `vaultAuth: {role: edge, createNamespace: false}`.
  - `values.schema.json`: `routes` minItems 1, CIDR pattern; `tags` minItems 1, pattern
    `^tag:`.
  - templates: `namespace.yaml` (hl-common), `auth-secret.yaml` (VSS → `tailscale-auth`, key
    `client_secret`, rollout restart of the Deployment), `rbac.yaml`, and `deployment.yaml`
    using the env set from T032. The auth key env is built in the container command as
    `TS_AUTHKEY="${CLIENT_SECRET}?ephemeral=false&preauthorized=true"`, so the suffix never
    sits in a Secret.
  - `ci/default-values.yaml` with `routes: ["10.0.0.100/32"]`.
  - `README.md.gotmpl`: purpose, the policy dependency (link to `docs/remote-access.md`), the
    state Secret, and any T032 deviation with its expiry condition.

  T031 now passes.
- [X] T034 [P] [US2] Write `hl-helm/docs/remote-access.md`:
  - how remote access works (with the diagram from the 2026-10-30 discussion, simplified);
  - the tailnet policy (copy of `feat/contracts/tailnet-policy.md`);
  - adding a device;
  - removing a lost device (*Machines → … → Remove*);
  - rotating the OAuth client secret (new secret → Vault → VSO restarts the pod);
  - the router-rebind fallback.

  Link it from `hl-helm/README.md` and `hl-helm/docs/access.md`.
- [X] T035 [US2] Add the `tailscale` entry to `hl-helm/bootstrap/install-order.yaml` after
  `ingress-tls` (contract §1). In `hl-helm/scripts/infra-status.sh`, print a "Tailscale" line:
  the pod phase and the last `Health` / `Running` message from its logs, never the auth key.
- [ ] T036 [US2] Run `hl-helm/scripts/validate.sh` until `GATE PASS`. Open the hl-helm PR "005:
  tailscale subnet router" with the rendered templates and the T032 results; merge after CI
  passes.
- [ ] T037 [US2] On the homelab: `bootstrap/install.sh --context homelab --only tailscale --diff`,
  owner review, then apply. Check that the pod is Ready and that *Machines* shows `homelab`
  connected with route `10.0.0.100/32` approved.
- [ ] T038 [US2] **MANUAL (owner)** (only if the route shows "awaiting approval"): quickstart
  **M7**. Approve it in *Machines → homelab → Edit route settings*. If this was needed, fix the
  `autoApprovers` block in the policy and note why in `feat/contracts/tailnet-policy.md`.
- [ ] T039 [US2] Run quickstart **V4** (owner's phone on mobile data; Tailscale on loads signed
  in within 30 s; Tailscale off fails), **V5**, **V6** (port scan from outside, router *Port
  forwarding* empty) and **V9** (restart; no duplicate machine). Record pass/fail per check in
  this task.

**Checkpoint**: HL Budget works from anywhere through the VPN, and nothing is exposed.

---

## Phase 5: User Story 3 — Give a future app a trusted address with no certificate work (Priority: P3)

**Goal**: An app gets a trusted address by choosing its name only. Proof: move `nginx`.

**Independent Test**: quickstart **V7**.

- [ ] T040 [US3] Move the reference app:
  - set `web.ingress.host: nginx.<domain>` in `hl-helm/charts/apps/nginx/values.yaml`, with the
    example comment `todo.<domain>`;
  - update the reference-chart guard `hl-common/placeholders`
    (`web.ingress.host=nginx.<domain>`) in **both** `hl-helm/charts/apps/nginx/Chart.yaml` and
    `hl-helm/charts/apps/hl-budget/Chart.yaml`;
  - update the duplicate-host case in `hl-helm/scripts/self-test.sh`;
  - bump both chart versions;
  - run `helm-docs`, `scripts/validate.sh` and `scripts/validate.sh --self-test` until both pass.
- [ ] T041 [P] [US3] Update `hl-helm/docs/onboarding.md` and `hl-helm/docs/access.md`:
  - app hosts are `<name>.<domain>`, one label;
  - leave `ingress.tls.secretName` unset (Traefik's default certificate is the shared
    wildcard);
  - no DNS step is needed (the wildcard record);
  - replace the hosts-file and `curl -k` instructions with the real names (keep
    `curl --resolve` as a debugging tip, without `-k`);
  - a private app needs nothing else; going public is a later feature (Cloudflare Tunnel).

  Update `hl-helm/charts/apps/nginx/README.md.gotmpl` if it mentions hosts.
- [ ] T042 [US3] Open and merge the hl-helm PR "nginx: move to nginx.<domain>; onboarding
  docs". Check that Argo CD shows `nginx` Synced/Healthy, then run quickstart **V7** from a home
  phone and the VPN-connected phone. Record the time from editing the host to the app loading
  trusted (SC-004: under 10 minutes, excluding Argo CD's sync delay).

**Checkpoint**: a second app proves the platform covers any new name.

---

## Phase 6: User Story 4 — Certificates keep working without attention (Priority: P3)

**Goal**: Renewal is automatic, and a failure is visible well before expiry.

**Independent Test**: quickstart **V8**.

- [ ] T043 [US4] Run quickstart **V8** on the homelab:
  - `hl-helm/scripts/infra-status.sh` shows `wildcard READY True` and the expiry date;
  - force a renewal (`cmctl renew -n ingress-tls wildcard`; if `cmctl` is not installed, delete
    the `wildcard-tls` Secret);
  - the Certificate returns to Ready with a later `notAfter`;
  - V2 shows the new end date;
  - the app showed no warning throughout.
- [ ] T044 [US4] Failure drill (owner present; quickstart V8 optional part):
  1. Write a deliberately wrong token to `kv/infra/cloudflare`, wait for VSO to sync (or restart
     the sync), and force a renewal.
  2. Confirm `infra-status.sh` shows `READY False` with the reason, and the site keeps serving
     the previous certificate.
  3. Restore the real token, renew again and confirm `READY True`.
  4. Record the result in this task.
- [ ] T045 [P] [US4] Add a "Certificates" section to `hl-helm/docs/bootstrap.md`:
  - what renews them and when (about 30 days before expiry for a 90-day certificate);
  - where to look (`infra-status.sh`);
  - how to rotate the Cloudflare token;
  - how to rebuild from scratch (cert-manager → ingress-tls with staging → production).

**Checkpoint**: all four stories done.

---

## Phase 7: Polish & Cross-Cutting Concerns

- [ ] T046 [P] Update `hl-helm/README.md`:
  - the tier table and the flowchart get cert-manager, ingress-tls and tailscale in the
    infrastructure tier;
  - "How a running application is wired" gets the wildcard certificate;
  - a "Cluster baseline" line for Traefik's default certificate and redirect.

  Update `hl-helm/docs/tiers.md` (members) and the Layout list in `hl-helm/CLAUDE.md` the same
  way. Open and merge a docs PR, or fold these into T042's PR if it is still open.
- [ ] T047 [P] Run quickstart **V10** (credential leak check) across `hl-helm/`, this repo and
  the tailscale pod logs, and record the result. Confirm `git -C hl-helm log -p` on the feature
  branch has no `tskey-` or token values.
- [ ] T048 Run `hl-helm/scripts/infra-status.sh` once more. Every release, including the three
  new ones, should be `deployed`, with no `DRIFT` and no unhealthy pods. Record the output in
  this task.
- [ ] T049 Update `feat/spec.md` **Status** to "Implemented and deployed <date>", naming the
  hl-helm PRs and the domain. Fill in the domain in `feat/quickstart.md`. Mark every task here
  `[X]` with its recorded outcome. Commit this feature folder on branch
  `005-home-and-remote-access` with Conventional Commit messages.
- [ ] T050 Open the small spec-repo PR that adds the 005 row to the README **Features** table
  ("Implemented <date>: HL Budget at https://budget.<domain> on any home device and over
  Tailscale away from home; shared wildcard certificate for all homelab apps").
- [ ] T051 Update the memory note `hl-budget-tls-vpn-deferred.md` (in the Claude project memory
  folder) and its `MEMORY.md` line:
  - the feature is done;
  - the new host and domain;
  - the old `hl-budget.home.example` is gone;
  - remote access is through Tailscale.

  Also update `homelab-kubeconfig.md`'s host line (the ingress host is now `budget.<domain>`;
  `curl -k` is no longer needed).
- [ ] T052 Remove the hl-helm worktree once all hl-helm PRs are merged:
  `git -C ../hl-helm worktree remove "$PWD/hl-helm"`. Keep the `.gitignore`/`.ignore` lines for
  next time.

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: T001 and T008 can start at once. The manual tasks follow their own chain:
  T002 → T004, T005; T003 → T006, T007.
- **Foundational (Phase 2)**: chart writing (T009–T017) needs only T001 and T008. The k3d
  rehearsal T018 needs T004 and T005. Applying (T020–T023) needs the merged PR T019 and T004.
  **Blocks every story.**
- **US1 (Phase 3)**: needs Phase 2 and T005. **This is the MVP.**
- **US2 (Phase 4)**: needs Phase 2 for the certificate and T006/T007 for Tailscale. Its chart
  work (T031–T036) can run in parallel with US1. Its live checks (T039) assume US1's
  hl-budget address is live.
- **US3 (Phase 5)**: needs Phase 2 only; independent of US1 and US2. V7's VPN half needs US2.
- **US4 (Phase 6)**: needs Phase 2 (T016's status output). Independent of the other stories.
- **Polish (Phase 7)**: after the stories that are being delivered.

### Within Each Phase

- Chart tests first (T010, T031), seen failing; then the chart; then `validate.sh`.
- Rehearse on k3d before applying to the homelab (T018, T032).
- Infrastructure: PR merged → `--diff` reviewed by the owner → apply from committed `main`.
- App charts: PR → merge → Argo CD. The hl-budget merge happens together with the Auth0 change
  (T026).

### Parallel Opportunities

- **Setup**: T002 ∥ T003 ∥ T008 (and T001). Then T004 ∥ T005, and T006 ∥ T007.
- **Phase 2 writing**: T010 ∥ T011 ∥ T012 ∥ T016, then T013 → T014 → T015 → T017.
- **Across stories**: once Phase 2 is applied, US1 (T024–T030), US2 chart work (T031–T036), US3
  (T040–T042) and US4 (T043–T045) can proceed in parallel. They touch different files, except
  that T040 and T024 both bump `charts/apps/hl-budget/Chart.yaml`, so merge one PR before
  rebasing the other.
- **Polish**: T046 ∥ T047.

---

## Parallel Example: Phase 2 chart writing

```text
Task: "T010 ingress-tls unit tests in hl-helm/charts/infrastructure/ingress-tls/tests/ingress_tls_test.yaml"
Task: "T011 Vault policy and role edge in hl-helm/bootstrap/vault-init.sh"
Task: "T012 cert-manager wrapper chart in hl-helm/charts/infrastructure/cert-manager/"
Task: "T016 certificate status in hl-helm/scripts/infra-status.sh"
```

## Parallel Example: after Phase 2 is live

```text
Track A (US1): T024 → T025 → T026 (owner + merge) → T027 → T028, T029 → T030
Track B (US2): T031 → T032 → T033, T034 → T035 → T036 → T037 → T038? → T039
Track C (US3): T040 → T041 → T042
```

---

## Implementation Strategy

### MVP First (User Story 1)

1. Phase 1: the owner buys the domain and creates the credentials (about 45 minutes), while the
   agent creates the worktree and confirms the redirect key.
2. Phase 2: charts, tests, k3d rehearsal, PR, then apply on the homelab (staging → production).
3. Phase 3 (US1): the hl-budget address switch with Auth0, then the router check.
4. **Stop and validate**: V1, V2, V3, V11. HL Budget now works on every home device; this alone
   fixes the main pain.

### Incremental Delivery

1. US1 → any home device works.
2. US2 → away from home over Tailscale.
3. US3 → nginx proves the platform; docs make future apps a one-line change.
4. US4 → renewal proven and visible.
5. Polish → docs, leak check, status, README row, memory.

### Notes

- Never write a secret value into a file, a commit, a PR, a task note or a log; record only that
  it was stored.
- `bootstrap/install.sh` refuses to apply from an uncommitted tree. Always pull the merged
  `main` into the worktree first.
- If the owner's SSH tunnel is down, cluster tasks wait; chart writing does not need it.
