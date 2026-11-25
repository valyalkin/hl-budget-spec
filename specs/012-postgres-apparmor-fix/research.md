# Research: Postgres AppArmor Fix

All findings come from the live cluster and node on 2026-11-25 and the hl-helm repository at
`origin/main` `d869256`.

## R1. How to stop AppArmor blocking Postgres's internal signals

**Decision**: Run the Postgres pods with AppArmor `Unconfined`, through the CloudNativePG
`Cluster` field `spec.podSecurityContext.appArmorProfile: {type: Unconfined}`.

**Rationale**:
- The denial is between two labels of the same container profile
  (`cri-containerd.apparmor.d` → `cri-containerd.apparmor.d//&unconfined`). With no profile
  attached there are no labels to mismatch.
- CNPG 1.30.1 exposes `podSecurityContext` on the `Cluster` (checked with
  `kubectl explain cluster.spec.podSecurityContext.appArmorProfile`), so no operator
  change, annotation or node change is needed.
- The pod-level field covers both containers in the pod (`postgres` and the
  `bootstrap-controller` init container).
- The remaining protections (non-root UID 26, all capabilities dropped, no privilege
  escalation, read-only root file system, seccomp `RuntimeDefault`) are untouched.

**Alternatives considered**:
- *Roll back to kernel 7.0.0-38*: it was removed from the node on 2026-11-25, and pinning an old
  kernel forgoes security fixes. Rejected.
- *Upgrade to 7.0.0-44*: tested by a reboot on 2026-11-25; it still fails (232 Postgres
  denials). Rejected.
- *Write a custom `Localhost` AppArmor profile that allows signals to the stacked label*: keeps
  AppArmor, but needs a profile file installed on the node outside Helm (breaks hl-helm I) and
  depends on the exact label the regression produces. Too much for a personal homelab
  (hl-helm VII). Can be revisited if the kernel stays broken.
- *Disable AppArmor for the whole node or for containerd*: affects every workload. Rejected
  (FR-003).
- *Legacy annotation `container.apparmor.security.beta.kubernetes.io/postgres: unconfined`*:
  deprecated since Kubernetes 1.30 in favour of the field. Rejected.

## R2. Where the switch lives and what its default is

**Decision**: New value `cluster.appArmorUnconfined` in the Postgres chart, default `false` in
`values.yaml`; the homelab sets it to `true` in `bootstrap/values/postgres.yaml`.

**Rationale**: hl-helm IV requires `values.yaml` defaults to be the safest option and
environment differences to live in values files. The homelab's kernel is that kind of
difference: a disposable k3d cluster on another kernel does not need the exemption. Since
`install.sh` always applies the Postgres chart with `bootstrap/values/postgres.yaml`, the
homelab gets the fix on every install and upgrade, which is what the spec's "default on"
(FR-004) is for. FR-004 is reworded accordingly: one value, on for the homelab.

**Alternatives considered**: default `true` in `values.yaml`, as first described in the spec.
It conflicts with hl-helm IV and would silently weaken any other cluster using the chart.
Rejected.

## R3. Keeping the operator's defaults when overriding the pod security context

**Decision**: When the value is on, render the full pod security context the operator applies
today, plus `appArmorProfile`:

| Field | Value | Source |
|---|---|---|
| `runAsNonRoot` | `true` | live pod `spec.securityContext` |
| `runAsUser` | `26` | live pod |
| `runAsGroup` | `26` | live pod |
| `fsGroup` | `26` | live pod |
| `seccompProfile.type` | `RuntimeDefault` | live pod |
| `appArmorProfile.type` | `Unconfined` | new |

**Rationale**: CNPG documents `podSecurityContext` as overriding the operator's default pod
security context, not merging with it. A missing `fsGroup` or `runAsUser` could leave Postgres
unable to read its data directory (spec edge case "Operator defaults drift"). The
container-level `securityContext` (capabilities, read-only root file system, privilege
escalation) is left alone so the operator keeps its defaults there.

**Alternatives considered**: setting only `appArmorProfile`. Rejected because it would drop
UID/GID/fsGroup and seccomp at pod level.

## R4. How the change reaches the live cluster

**Decision**:
1. Unseal Vault.
2. Commit on hl-helm branch `012-postgres-apparmor-fix`.
3. Run `bootstrap/install.sh --only postgres --diff`; the owner reviews it. The expected diff is
   the `podSecurityContext` block only.
4. Run `bootstrap/install.sh --only postgres`.
5. Merge the PR afterwards.

**Rationale**: hl-helm I and III require infrastructure changes to come from a committed chart,
via the script, after a reviewed diff. The chart's `pre-upgrade` provision Job logs in to Vault,
so Vault must be unsealed first (it is sealed after this morning's reboot). Helm's last apply
(revision 2, 2026-10-25, chart 0.2.0) matches `origin/main`: no chart, values or `hl-common`
commits since. So the diff should contain only this change.

**Fallback**: if Vault cannot be unsealed promptly, apply the same block with
`kubectl patch cluster postgres -n postgres --type merge` to end the outage. Then reconcile with
`install.sh` in the same working session, as hl-helm I requires. Helm's three-way merge adopts
the identical field without churn.

**Alternatives considered**: merge first, then apply from `main`. Equivalent but slower; the
outage is ongoing, and applying from the reviewed branch commit is allowed.

## R5. What happens to the running pod

**Decision**: Let the operator roll the pod. CNPG detects the pod spec change and recreates
`postgres-1` on the same PVC (single instance, so an in-place restart rather than a switchover).
Postgres then replays the WAL again and, with signals allowed, writes the end-of-recovery
checkpoint.

**Rationale**: The log already shows `redo done at 0/96000110`, so the WAL is intact and
recovery fails only at the checkpoint signal. The PVC is `local-path` with
`helm.sh/resource-policy: keep` on the Cluster, and nothing in this change touches storage
(FR-007).

**Risk and mitigation**: if recovery fails for a different reason after the change, stop and
investigate; never delete the pod's PVC or the Cluster.

## R6. Testing the chart

**Decision**:
- Add `charts/infrastructure/postgres/tests/cluster_test.yaml` (helm-unittest) with three cases:
  - the value on renders `appArmorProfile: Unconfined` plus the five protections (SC-006, pod
    level);
  - the value off renders no `podSecurityContext`;
  - the schema rejects a non-boolean.
- Extend `scripts/validate.sh` to run `helm unittest` for every chart that ships a `tests/`
  directory.
- Add `ci/apparmor-unconfined-values.yaml` so the "on" case is rendered and schema-validated by
  kubeconform against the CNPG CRD catalog.

**Rationale**: the existing `tests/unit` harness only exercises `hl-common`; infrastructure charts
have no unit tests yet. Touching the gate needs `scripts/validate.sh --self-test` afterwards
(hl-helm CLAUDE.md).

**Alternatives considered**: render and grep in a shell step. Weaker and inconsistent with the
repository's helm-unittest convention.
