# Feature Specification: Home and Remote Access

**Feature Branch**: `005-home-and-remote-access`

**Created**: 2026-10-30

**Status**: Implemented and deployed 2026-10-31. Domain `valyalkin.com`; HL Budget at https://budget.valyalkin.com. hl-helm PRs #21 (cert-manager, ingress-tls, gates), #22 (production certificates), #23 (hl-budget host), #24 (Tailscale gateway), #25 (nginx host and docs).

**Input**: User description: "Enable my application to be accessible from home network on any device and also from outside using any VPN solution which is easiest to setup and secure." Refined in discussion on 2026-10-30: use an own domain bought on Cloudflare, one trusted certificate shared by every homelab app, and Tailscale for access from outside; no ports opened to the internet. The owner is the primary user; the home router is a standard consumer router with unknown settings. Buying the domain and the other account and device steps are manual tasks for the owner.

## Scope

Today HL Budget is reachable only at `https://hl-budget.home.example`, which works on a device
only after its hosts file is edited, and every browser shows a certificate warning. Phones and
tablets cannot use it at all.

This feature makes HL Budget reachable at one stable address, `https://budget.<domain>`, with a
trusted certificate:

- **At home**: from any device on the home network (phone, tablet, laptop), with nothing
  installed or configured on the device.
- **Away from home**: from the owner's devices over a private VPN, using the same address.

It also turns the domain and certificate into a shared homelab capability, so the owner's future
apps (for example a personal page at `https://<domain>`) get a trusted address by choosing a
name, with no certificate work of their own.

`<domain>` is a placeholder for the domain the owner buys as the first manual step; it is
filled in once known. No application code or API changes are expected; the work is in the
homelab platform, the HL Budget deployment configuration and the sign-in provider settings.

### Decisions from the discussion (2026-10-30)

| Topic | Decision | Why |
|---|---|---|
| Name | Own domain registered with Cloudflare; apps use flat subdomains (`budget.<domain>`), the bare domain is kept for a future personal page | Real DNS works on every device; flat names keep one wildcard certificate valid for all apps |
| Certificate | One trusted certificate for `<domain>` and `*.<domain>`, issued and renewed automatically by proving domain ownership through DNS | Nothing has to be reachable from the internet to get or renew it |
| Home access | The public DNS name points at the homelab node's private address | Works on any home device with no per-device setup |
| Remote access | Tailscale, with the homelab node acting as the gateway to the homelab address only | Outbound-only connections, no port forwarding, free for personal use, apps on all platforms |
| Going public later | Out of scope; the expected path is a Cloudflare Tunnel on the same hostnames | Choosing stable names now avoids renaming later |

## Clarifications

### Session 2026-10-30

- Q: Who uses the app? → A: The owner, primarily. Other household members are not set up in
  this feature.
- Q: What router is at home? → A: A standard consumer router; details unknown. The spec
  therefore includes a fallback if the router refuses DNS answers that point at a private
  address (Edge Cases).
- Q (during implementation): With the VPN on, a device sends all homelab traffic through the
  VPN, even at home, and the policy allows web only, so the owner's SSH to the node fails while
  the VPN is on. Allow SSH through the VPN? → A: No; too risky. The VPN stays web-only (80, 443).
  At home, the owner turns off the VPN's subnet routes (or the VPN) on the laptop before using
  SSH or `kubectl` (Edge Cases).
- Q: Branch name? → A: `005-home-and-remote-access` (the earlier empty `005-tailscale-access`
  branch was renamed), since the feature covers the domain and certificate as well as the VPN.

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Use HL Budget from any device at home (Priority: P1)

The owner picks up their phone, tablet or laptop on the home Wi-Fi, opens
`https://budget.<domain>`, sees no security warning, signs in and uses HL Budget as normal.
No hosts-file edits, certificates or apps are installed on the device.

**Why this priority**: it is the most frequent use and fixes today's main pain: phones cannot
reach the app and every browser warns.

**Independent Test**: on a phone that has never been configured for the homelab, join the home
Wi-Fi, open the address, sign in and add a spending.

**Acceptance Scenarios**:

1. **Given** a phone on the home Wi-Fi with no homelab setup, **When** the owner opens
   `https://budget.<domain>`, **Then** the HL Budget page loads with the browser showing a
   secure connection and no warning.
2. **Given** the owner is on that page, **When** they sign in, **Then** sign-in completes and
   returns them to HL Budget on the same address, signed in.
3. **Given** a laptop that previously used `hl-budget.home.example` through its hosts file,
   **When** the owner removes that hosts entry and opens the new address, **Then** the app works
   the same as on the phone.

---

### User Story 2 - Use HL Budget away from home over a VPN (Priority: P2)

Away from home (mobile data, office or hotel Wi-Fi), the owner turns on the VPN app on their
phone or laptop and opens the same `https://budget.<domain>` address. The app works exactly as
it does at home. With the VPN off, the address does not reach the app.

**Why this priority**: it is the second goal of the feature, but it depends on the address and
certificate from Story 1.

**Independent Test**: on a phone on mobile data (Wi-Fi off), connect the VPN, open the address,
sign in and view this month's spendings; then disconnect the VPN and confirm the address no
longer loads.

**Acceptance Scenarios**:

1. **Given** the owner's phone on mobile data with the VPN connected, **When** they open
   `https://budget.<domain>`, **Then** the app loads with no warning and sign-in works.
2. **Given** the same phone with the VPN disconnected, **When** they open the address, **Then**
   the app is not reachable.
3. **Given** a device that is not one of the owner's VPN devices, **When** anyone on the
   internet tries to reach the homelab, **Then** nothing is reachable: no ports are open on the
   home router.
4. **Given** the owner's device is connected to the VPN, **When** it tries to reach other
   devices on the home network (other than the homelab), **Then** they are not reachable through
   the VPN.

---

### User Story 3 - Give a future app a trusted address with no certificate work (Priority: P3)

When the owner deploys another app in the homelab, such as a personal page on `<domain>` or the
existing `nginx` test app at `nginx.<domain>`, they only choose its name. The app is
immediately served with the same trusted certificate, at home and over the VPN.

**Why this priority**: it is the reason for owning a domain rather than using a VPN-only name,
but HL Budget does not depend on it.

**Independent Test**: move the existing `nginx` app to `nginx.<domain>` by changing only its
hostname (plus its DNS name), then open it from a home phone and a VPN-connected phone with no
warning.

**Acceptance Scenarios**:

1. **Given** the shared certificate is in place, **When** an app is given a new name under
   `<domain>` and its DNS name is added, **Then** it is served with a trusted certificate without
   the app requesting or storing a certificate of its own.
2. **Given** the shared certificate, **When** an app is served on the bare `<domain>`, **Then**
   it is also trusted.

---

### User Story 4 - Certificates keep working without attention (Priority: P3)

Months later, the owner opens HL Budget and it still shows a trusted connection. The owner never
renewed anything by hand. If renewal ever fails, the owner can see that in the homelab's normal
status views well before the certificate expires.

**Why this priority**: an expired certificate brings back the warnings and can break sign-in,
but it only matters after the first weeks.

**Independent Test**: confirm that renewal is automatic by forcing an early renewal, and check
that the current certificate's expiry date and renewal health are visible in the homelab's status
views.

**Acceptance Scenarios**:

1. **Given** the certificate is within its renewal window, **When** renewal runs, **Then** a new
   certificate is in use with no downtime and no manual action.
2. **Given** renewal fails (for example the DNS provider credential is revoked), **When** the
   owner looks at the homelab status, **Then** the failure is visible while the current
   certificate still has at least 2 weeks of validity left.

---

### Edge Cases

- **The home router refuses DNS answers that point at a private address** (DNS rebind
  protection): home devices cannot resolve `budget.<domain>`. The quickstart must give a check
  for this and two fixes: add an exception for `<domain>` in the router, or, if the router has no
  such setting, keep the VPN app on at home on that device (it resolves names on its own and
  still reaches the app).
- **The internet is down at home**: home devices cannot look up the public name. The app is
  unreachable by name until the internet returns; this is accepted for this feature. Sign-in
  needs the internet anyway.
- **The VPN is connected while at home**: the app still works at the same address. It does not
  matter whether traffic goes directly or through the VPN. Only web traffic is allowed through the VPN, so SSH to the node
  (and the `kubectl` tunnel) fails while the VPN routes are on. The owner turns off "Use
  Tailscale subnets" (or Tailscale) on the laptop for terminal work at home.
- **The homelab node's private address changes** (for example after a router reset): the app is
  unreachable until the DNS record and the VPN route are updated. The quickstart must list both
  places; reserving the node's address in the router is a recommended manual step.
- **A VPN device is lost or stolen**: the owner can remove it from the VPN in one place, and its
  access ends immediately. HL Budget sign-in still protects the data either way.
- **The old address `hl-budget.home.example`** stops being used: sign-in no longer accepts it,
  and the hosts-file entries on the owner's laptops are removed (manual step).
- **The domain registration lapses**: names and certificate renewal break. Auto-renew is a
  required manual setting.
- **A DNS provider credential leaks**: it can change only the DNS records of `<domain>`, not the
  account, and it can be revoked and replaced from Vault without other changes.

## Requirements *(mandatory)*

### Functional Requirements

**Address and certificate**

- **FR-001**: HL Budget MUST be served at `https://budget.<domain>`, with the web app at `/` and
  the API under `/api` on the same address, as today.
- **FR-002**: The address MUST present a certificate that current browsers on iOS, Android,
  macOS and Windows trust with no setup on the device.
- **FR-003**: One certificate MUST cover `<domain>` and every first-level name under it
  (`*.<domain>`), and MUST be shared by all homelab apps. An app MUST be able to use it by
  choosing a name, without requesting or storing a certificate of its own.
- **FR-004**: The certificate MUST be obtained and renewed automatically, without anything in
  the homelab being reachable from the internet, and renewed at least 2 weeks before it expires.
- **FR-005**: The current certificate's expiry date and any renewal failure MUST be visible
  through the homelab's existing status tools (the infrastructure status report the owner
  already runs, and the cluster resources themselves).
- **FR-006**: Plain `http://` requests to any app address MUST redirect to `https://`.

**Home access**

- **FR-007**: The name `budget.<domain>` MUST resolve, through public DNS, to the homelab node's
  private address, so any device on the home network reaches the app directly with no device
  setup.
- **FR-008**: The DNS records for `<domain>` MUST be kept in the owner's Cloudflare account; the
  credential the homelab uses MUST be limited to editing DNS records of `<domain>` only.

**Remote access**

- **FR-009**: The owner's devices MUST reach the app away from home through a private VPN,
  using the same address and the same certificate as at home.
- **FR-010**: Remote access MUST NOT require opening or forwarding any port on the home router,
  and the homelab MUST NOT accept any connection from the public internet.
- **FR-011**: Through the VPN, devices MUST reach only the homelab node's address, not the rest
  of the home network.
- **FR-012**: Only devices the owner has added to their VPN account MUST be able to connect; the
  owner MUST be able to remove a device in one place, ending its access.
- **FR-013**: The homelab's VPN gateway MUST stay connected without the owner signing in again
  periodically (its credential must not expire every few months), and MUST reconnect by itself
  after the node restarts.

**Sign-in and the old address**

- **FR-014**: Sign-in, sign-out and session cookies MUST work on the new address; the sign-in
  provider MUST accept the new address and MUST no longer accept `hl-budget.home.example`.
- **FR-015**: The old address `hl-budget.home.example` MUST be retired from the HL Budget
  deployment once the new address is verified.

**Platform and secrets**

- **FR-016**: The domain certificate and the VPN gateway MUST be shared homelab platform pieces
  (not part of the HL Budget chart), recorded in the homelab repository and applied the way
  that repository applies platform components (reviewed, from a committed state). HL Budget's
  own changes keep flowing through the normal pull request and Argo CD flow.
- **FR-017**: The DNS provider credential and the VPN gateway credential MUST be stored only in
  Vault and delivered to the cluster from there; they MUST NOT appear in git, Helm values,
  images or logs.
- **FR-018**: HL Budget's address MUST be set only in its deployment configuration (one values
  file), so a later rename or public exposure changes configuration only, not code or images.

**Manual steps and documentation**

- **FR-019**: The steps only the owner can do MUST be written down in the quickstart and appear
  as explicit manual tasks in the task list:
  1. Create a Cloudflare account with two-factor authentication, buy `<domain>` with auto-renew
     on, and confirm it is active.
  2. Create a Cloudflare API token limited to DNS edits on `<domain>`, and store it in Vault.
  3. Create a Tailscale account with two-factor authentication, create the gateway's auth key,
     and store it in Vault.
  4. Approve the homelab gateway's route to the node's address in the Tailscale admin console,
     if it is not approved automatically.
  5. Install Tailscale on the owner's phone and laptop and sign in.
  6. Reserve the homelab node's address in the router (recommended), and check the router's DNS
     rebind behaviour; add an exception for `<domain>` if needed.
  7. Update the HL Budget application in Auth0 with the new address and remove the old one.
  8. Remove the `hl-budget.home.example` hosts-file entries from the owner's laptops.
- **FR-020**: The quickstart MUST include a short check that tells the owner whether their router
  blocks private-address DNS answers, and what to do if it does.

### Key Entities

- **Domain**: the owner's registered domain (`<domain>`), its registration and its DNS records.
  Owned by the owner in Cloudflare.
- **App address**: a name under the domain (`budget.<domain>`, later `<domain>`,
  `nginx.<domain>`), pointing at the homelab node.
- **Shared certificate**: one trusted certificate for `<domain>` and `*.<domain>`, renewed
  automatically and used by every app address.
- **VPN device**: one of the owner's devices allowed into their private VPN; can be added and
  removed by the owner.
- **Homelab gateway**: the homelab's own VPN member that gives VPN devices a route to the homelab
  node's address only.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: From a phone, a tablet and a laptop on the home Wi-Fi, each with no homelab setup,
  HL Budget opens with no security warning and the owner can sign in and add a spending: 3 of 3
  devices succeed.
- **SC-002**: Away from home with the VPN on, the owner reaches HL Budget at the same address and
  signs in within 30 seconds of connecting the VPN; with the VPN off, the address does not load.
- **SC-003**: A scan of the home's public internet address shows no open ports added by this
  feature.
- **SC-004**: A second app (the existing `nginx` app) gets a trusted address by changing only its
  name and adding a DNS name, in under 10 minutes and with no certificate steps.
- **SC-005**: Over the first 90 days the certificate renews at least once with no manual action
  and no visible downtime.
- **SC-006**: The owner completes all manual steps from the quickstart in under 1 hour, without
  needing to ask for help.
- **SC-007**: No credential for DNS or VPN is found in any git repository, image or log.

## Assumptions

- The owner is the only user for now. Sharing with household members (inviting them to the VPN
  and adding them in the sign-in provider) can come later and is out of scope.
- The homelab is one node at `10.0.0.100`, on a home network that uses private addresses, with
  the existing ingress that terminates TLS (as stated in the constitution).
- The owner is willing to pay for a domain (roughly $10–12 a year) and use free plans of
  Cloudflare and Tailscale; these plans cover a single owner's devices.
- Publishing a private address in public DNS is accepted: it reveals only that a homelab exists
  at a private address, which is not reachable from the internet.
- The existing `nginx` test app is moved to `nginx.<domain>` as the proof for Story 3; its old
  name is retired too.
- No HL Budget application code changes are expected. The address is configuration (the
  deployment's base URL and ingress host) and the sign-in provider's allowed addresses.
- Putting any app on the public internet is out of scope; the stable names chosen here allow
  doing it later (expected through a Cloudflare Tunnel, with an extra access check in front)
  without renaming.
- Remote access to other homelab tools (Argo CD, Vault) is not a goal of this feature; they stay
  as they are today.

## Dependencies

- Features 001–003 deployed: HL Budget running in the homelab with Auth0 sign-in.
- The owner's Cloudflare and Tailscale accounts and the domain purchase (manual steps, FR-019).
- The hl-helm repository and Argo CD for platform and app changes; Vault for the two new
  credentials.
