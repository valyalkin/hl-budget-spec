# Feature Specification: Postgres AppArmor Fix

**Feature Branch**: `012-postgres-apparmor-fix`

**Created**: 2026-11-25

**Status**: Implemented and deployed 2026-11-25. hl-helm PR #36 (chart `postgres` 0.3.0, release revision 3). HL Budget restored at 10:47 EST after about 42 hours down.

**Input**: User description: "Restore the homelab Postgres (and with it HL Budget) after a kernel AppArmor regression. Problem: since 2026-11-23 the homelab node runs Ubuntu kernel 7.0.0-43/44. After an unclean power loss, the CloudNativePG pod postgres-1 (namespace postgres) must do crash recovery; at the end of recovery the Postgres startup process signals the checkpointer and the kernel's AppArmor denies it ("could not signal for checkpoint: Permission denied"; kernel log: apparmor="DENIED" operation="signal" profile="cri-containerd.apparmor.d" peer="cri-containerd.apparmor.d//&unconfined"), because runc's profile is now "converted to stacking" and processes in one container carry different labels. Postgres crashloops (400+ restarts), hl-budget-backend fails at Flyway and crashloops, the API returns 503 and the web app shows React error #441 "We couldn't load your spendings". Down ~42h, unnoticed. Kernel 7.0.0-38 had 0 Postgres denials; 7.0.0-44 (booted 2026-11-25) still fails. Solution: run the Postgres pod with AppArmor Unconfined via CNPG 1.30 Cluster.spec.podSecurityContext.appArmorProfile {type: Unconfined}, repeating the operator's default pod security context (runAsNonRoot, runAsUser/runAsGroup/fsGroup 26, seccomp RuntimeDefault) and keeping container hardening (non-root, drop ALL caps, read-only root fs, seccomp). Change it in hl-helm charts/infrastructure/postgres (values-driven, default on) and apply it to the live cluster manually (infrastructure tier is not managed by Argo CD). Success: postgres-1 completes recovery and is Ready, survives an unclean restart, hl-budget-backend Ready, budget.valyalkin.com API stops returning 503 and the Spendings screen loads, no data lost. Out of scope (follow-ups): crashloop alerting, planned reboots after kernel updates, Vault auto-unseal, reporting the kernel regression upstream."

## Scope

This is an incident fix in the homelab platform. No HL Budget application code or API changes;
the work is in the shared Postgres deployment (hl-helm `charts/infrastructure/postgres`) and its
live state on the cluster.

### What happened (incident summary, 2026-11-23 → 2026-11-25)

| When | Event | Evidence |
|---|---|---|
| 2026-11-11 | Unattended upgrades installed kernel 7.0.0-43; the node kept running 7.0.0-38 | apt history |
| 2026-11-23 15:14 | The node went down uncleanly (no shutdown sequence in the log) and came back at 16:22 on 7.0.0-43 | `journalctl --list-boots`, last lines of the previous boot |
| from 2026-11-23 16:22 | Postgres needs crash recovery. At its end the startup process signals the checkpointer; AppArmor denies the signal and Postgres exits. 400+ restarts | `could not signal for checkpoint: Permission denied`; kernel `apparmor="DENIED" operation="signal" comm="postgres" … peer="cri-containerd.apparmor.d//&unconfined"` |
| same | The HL Budget backend cannot reach the database, fails at start-up and crashloops; the API answers 503; the web app shows "We couldn't load your spendings" (React error #441) | backend logs, `no available server` from the ingress |
| 2026-11-25 10:22 | Rebooted cleanly into kernel 7.0.0-44 to test whether the newer kernel fixes it: it does not | same Postgres error and denials on 7.0.0-44 |

Root cause: on kernels 7.0.0-43 and later, the container runtime's AppArmor profile is applied
as a *stack*, so processes inside one container can carry two different labels
(`cri-containerd.apparmor.d` and `cri-containerd.apparmor.d//&unconfined`). The profile allows
signals only between identical labels, so Postgres processes can no longer signal each other.
Signal denials per boot: 28 on 7.0.0-38 (none from Postgres), 53,378 on 7.0.0-43 (48,628 from
Postgres). The runc profile, the AppArmor package and k3s did not change in that window. Other
workloads log the same kind of denial for non-essential signals and keep running.

The crash was the trigger, not the cause: any Postgres start that needs recovery fails on these
kernels.

### Decisions (2026-11-25)

| Topic | Decision | Why |
|---|---|---|
| Fix | Run the Postgres pods without an AppArmor profile (`Unconfined`); keep every other container protection | Removes the label mismatch at its source; works on the current and newer kernels; no node changes |
| Scope of the exemption | Postgres pods only, not the whole node or other workloads | Other workloads are not failing; keep the reduction in hardening as small as possible |
| Where | hl-helm Postgres infrastructure chart, controlled by one value (on in the homelab's values file, off by default in the chart), applied to the live cluster with hl-helm's install script | The infrastructure tier is not deployed by Argo CD; the chart must match what runs so the next upgrade keeps the fix |
| Kernel rollback | Rejected | The last working kernel (7.0.0-38) has been removed from the node, and pinning an old kernel forgoes security updates |

## User Scenarios & Testing *(mandatory)*

### User Story 1 - HL Budget works again (Priority: P1)

The owner opens HL Budget and sees their spendings, income and overview as before the outage,
with all their data intact.

**Why this priority**: The app has been unusable since 2026-11-23; this is the outage itself.

**Independent Test**: After the fix is applied to the live cluster, open
https://budget.valyalkin.com, sign in and open Spendings, Income and Overview for a month that
had entries before 2026-11-23.

**Acceptance Scenarios**:

1. **Given** the database has been crash-looping since 2026-11-23, **When** the fix is applied,
   **Then** the database finishes recovery and reports ready, without losing data.
2. **Given** the database is ready, **When** the backend next starts, **Then** it reports ready
   and the API stops answering 503.
3. **Given** the backend is ready, **When** the owner opens the Spendings screen, **Then** it
   shows the month's entries instead of "We couldn't load your spendings".
4. **Given** the app works, **When** the owner compares a month's totals with those before the
   outage, **Then** they match (no lost or duplicated records).

---

### User Story 2 - The database survives the next unclean restart (Priority: P2)

The next power cut or crash of the node does not take HL Budget down again: the database
recovers on its own once the node is back.

**Why this priority**: The fault is latent; without this the same outage repeats on the next
power cut.

**Independent Test**: With the fix in place, force an unclean stop of the database (kill its
pod without a graceful shutdown) and confirm it recovers and becomes ready without manual help.

**Acceptance Scenarios**:

1. **Given** the fix is in place, **When** the database is stopped uncleanly and restarted,
   **Then** it completes recovery and is ready within 2 minutes, with no "Permission denied"
   signal errors.
2. **Given** the fix is in place, **When** the node itself is rebooted, **Then** the database
   and the HL Budget backend come back ready without manual steps (Vault unseal aside).

---

### User Story 3 - The fix is recorded where the platform is defined (Priority: P3)

The owner can rebuild or upgrade the Postgres infrastructure from hl-helm and get the same
working configuration, and can see why the exemption exists and how to undo it.

**Why this priority**: Without it, the next manual upgrade of the Postgres chart silently drops
the fix and the outage returns.

**Independent Test**: Render the Postgres chart with default values and confirm it produces the
same setting that runs on the cluster; render it with the setting turned off and confirm the
operator's defaults come back.

**Acceptance Scenarios**:

1. **Given** the hl-helm change is merged, **When** the chart is rendered with default values,
   **Then** the result matches the live configuration.
2. **Given** a future kernel fixes the problem, **When** the owner turns the setting off,
   **Then** the chart goes back to the operator's default protections with no other edits.
3. **Given** someone reads the chart, **When** they look at the setting, **Then** a comment
   names the kernel regression, this spec, and how to check whether it is still needed.

### Edge Cases

- **Recovery from the failed state**: the database has crash-looped 400+ times mid-recovery.
  Applying the fix must let it finish that same recovery; it must not need a restore from
  backup or a fresh database. If recovery still fails for a different reason, stop and
  investigate before anything that could discard data.
- **Vault sealed**: after every node reboot Vault is sealed. Existing credentials keep working,
  so the fix must not depend on Vault being unsealed; unsealing stays a separate manual step.
- **Operator defaults drift**: overriding the pod's security settings replaces all of the
  operator's defaults, not just the AppArmor part. Every default the pod runs with today (user,
  group, file-system group, non-root, seccomp) must be kept, or the database may fail to read
  its data files.
- **Other workloads**: other pods log the same denials for non-essential signals. They are not
  changed by this feature; if one starts failing, it is a separate incident.
- **The backend is in a long restart back-off**: once the database is ready the backend may take
  up to its back-off interval (about 5 minutes) to retry; restarting it is allowed to shorten
  the outage.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: The Postgres pods MUST run without an AppArmor profile, so that Postgres processes
  in one pod can signal each other on kernel 7.0.0-43 and later.
- **FR-002**: The Postgres pods MUST keep every other protection they have today: non-root user
  and group 26, file-system group 26, seccomp runtime default, no privilege escalation, all
  capabilities dropped, read-only root file system.
- **FR-003**: The exemption MUST apply only to the Postgres pods; no other workload and no node
  setting changes.
- **FR-004**: The exemption MUST be controlled by a single chart value, turned on for the homelab
  in its values file. The chart's own default stays off, the safest option (hl-helm constitution
  IV; plan research R2). The homelab can turn it off without other edits once a fixed kernel is
  confirmed.
- **FR-005**: The chart MUST carry a comment naming the kernel regression, this spec, and the
  check that shows whether the exemption is still needed (Postgres signal denials in the kernel
  log).
- **FR-006**: The fix MUST be applied to the live cluster and then recorded in hl-helm through a
  pull request, so the chart and the cluster agree.
- **FR-007**: Applying the fix MUST NOT delete, recreate or restore the database or its storage;
  the existing data volume is recovered in place.
- **FR-008**: After the fix, the HL Budget backend MUST report ready and the API MUST stop
  answering 503 without any change to the backend or frontend.
- **FR-009**: The hl-helm documentation MUST record the incident's lesson for operators: after an
  unclean restart, check the database and the apps that depend on it, and how to read the
  AppArmor denial in the kernel log.
- **FR-010**: The follow-ups (crash-loop alerting, planned reboots after kernel updates, Vault
  auto-unseal, reporting the kernel regression upstream) MUST be listed in the spec's
  Out of scope with enough detail to become their own features.

### Key Entities

- **Postgres cluster**: the single shared database instance in the homelab (`postgres-1`),
  holding HL Budget's data and other apps' databases; its security settings are what this
  feature changes.
- **Security exemption setting**: the chart value that turns the AppArmor exemption on or off.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: Within 15 minutes of the fix being applied, the owner can sign in and see their
  spendings for the current month.
- **SC-002**: No HL Budget record created before the outage is missing or duplicated afterwards:
  month totals for October and November 2026 match the values before 2026-11-23.
- **SC-003**: After a forced unclean stop, the database is ready again within 2 minutes with no
  manual action, in 1 of 1 test runs, and its log shows no signal permission errors.
- **SC-004**: After a full node reboot, HL Budget works again with no manual step other than the
  Vault unseal, which it does not need in order to serve existing data.
- **SC-005**: Rendering the chart with defaults gives exactly the configuration running on the
  cluster: zero differences in the database's security settings.
- **SC-006**: Every other database protection listed in FR-002 is still in force after the
  change: 6 of 6 checks pass on the running pod.

## Assumptions

- The kernel regression is in the Ubuntu 7.0.0-43/44 kernels' AppArmor stacking and is not
  fixed by any package available on 2026-11-25; turning AppArmor off for Postgres only is an
  acceptable trade-off at personal scale (Constitution VI), since the container keeps its
  non-root user, dropped capabilities, read-only file system and seccomp filter.
- The installed database operator (CloudNativePG 1.30.1) lets a cluster set its pod security
  settings, including the AppArmor profile; the database is one instance with one data volume.
- The data volume is intact: recovery replays the write-ahead log to the end
  (`redo done`) and fails only at the final checkpoint signal.
- Infrastructure charts are applied by hand, not by Argo CD (hl-helm `docs/bootstrap.md`), so a
  live change is not reverted by automation.
- The owner holds the Vault unseal key and unseals it separately after reboots.

## Implementation notes (2026-11-25)

- **Applying the fix.** After `install.sh --only postgres` updated the `Cluster`, CloudNativePG
  did not roll the pod, because it cannot read the status of an instance that is down ("Cannot
  extract Pod status"). Deleting the crash-looping pod (not the PVC or Cluster) let the operator
  recreate it from the new spec on the same volume. Recovery then finished in 16 s
  (`redo done` → `end-of-recovery` checkpoint → ready), and no data was replayed beyond the WAL
  already on disk.
- **Unclean restart (US2).** A forced kill recovered in 13 s with 0 AppArmor signal denials. The
  backend's connection pool reconnected without restarting the pod.
- **Vault.** The first unseal failed with "message authentication failed" because the script used
  the shell's default kube context, the local k3d test cluster. It succeeded with
  `KUBECONFIG=~/.kube/homelab.yaml`. hl-helm `docs/bootstrap.md` now says so.
- **Chart default.** Following hl-helm constitution IV, the chart default is `false` and the homelab
  turns it on in `bootstrap/values/postgres.yaml` (plan research R2; FR-004 reworded).
- **Not yet observed:** a full node reboot with the fix in place (SC-004). The next real reboot is
  the check.

## Out of scope (follow-ups)

- **Crash-loop alerting**: the outage went unnoticed for about 42 hours. A follow-up feature
  should alert the owner when a pod in the homelab keeps restarting or an app's readiness fails.
- **Planned reboots after kernel updates**: unattended upgrades install kernels but never
  reboot, so a change sat unused from 2026-11-11 until an unplanned reboot. A follow-up should
  reboot on a schedule (or notify) so kernel changes surface at a known time.
- **Vault auto-unseal**: every reboot leaves Vault sealed until the owner runs the unseal
  script.
- **Reporting the kernel regression**: file an Ubuntu kernel/AppArmor bug with the denial lines
  and kernel versions, and turn the exemption off (FR-004) once a fixed kernel is confirmed.
- **Other workloads' denials**: non-essential signal denials in other pods are noted, not fixed.
