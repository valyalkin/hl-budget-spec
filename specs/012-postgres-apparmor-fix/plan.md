# Implementation Plan: Postgres AppArmor Fix

**Branch**: `012-postgres-apparmor-fix` | **Date**: 2026-11-25 | **Spec**: [spec.md](spec.md)

**Input**: Feature specification from `specs/012-postgres-apparmor-fix/spec.md`

## Summary

Postgres cannot finish crash recovery on kernels 7.0.0-43 and later because AppArmor blocks
signals between its own processes, and HL Budget is down with it. The fix runs the Postgres pods
with AppArmor `Unconfined` by setting the CloudNativePG `Cluster`'s pod security context, while
repeating every default the operator applies today. It lands in hl-helm's Postgres
infrastructure chart behind one value, `cluster.appArmorUnconfined` (chart default `false`,
turned on in the homelab's values file), and reaches the cluster through hl-helm's own gate:
`bootstrap/install.sh --only postgres --diff`, human review, then apply from the committed
branch. No HL Budget backend or frontend changes.

## Technical Context

**Language/Version**: Helm 4.1.1 charts (YAML + Go templates); Kubernetes 1.36 (k3s v1.36.5)

**Primary Dependencies**: CloudNativePG operator 1.30.1 (`Cluster.spec.podSecurityContext` with
`appArmorProfile`), hl-helm `hl-common` library chart 0.1.x (unchanged)

**Storage**: the existing CNPG cluster `postgres` (one instance, 5Gi `local-path` volume
`postgres-1`); kept and recovered in place

**Testing**: hl-helm gate `scripts/validate.sh` (helm lint, template + kubeconform, helm-unittest,
policy checks, helm-docs), a new helm-unittest suite for the Postgres chart, a `ci/` fixture
rendering the value off, `helm diff upgrade` reviewed before apply, then live checks (quickstart)

**Target Platform**: single-node k3s homelab, Ubuntu 26.04, kernel 7.0.0-44-generic

**Project Type**: infrastructure configuration (Helm chart change in the hl-helm repository)

**Performance Goals**: the database is ready within 2 minutes of an unclean stop (SC-003)

**Constraints**: no data loss and no restore (FR-007); every other pod and container protection
kept (FR-002); the change applies to the Postgres pods only (FR-003); Vault must be unsealed for
the apply because the chart's `pre-upgrade` provision Job logs in to Vault

**Scale/Scope**: one chart (template, values, schema, README), one values file, one test suite,
one CI fixture, one doc section; one live `helm upgrade`

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

Two constitutions apply: HL Budget's (this repository) and hl-helm's
(`hl-helm/.specify/memory/constitution.md`), because all changes land in hl-helm.

| Principle | Status | Notes |
|---|---|---|
| HL Budget I. Spec-driven | ✅ | This spec, plan and tasks precede the change |
| HL Budget II. Separate services, contract-first | ✅ | No service or API change |
| HL Budget III. Secure by default | ✅ | No secrets touched; the provision Job keeps writing only absent credentials |
| HL Budget IV. Tests and gates | ✅ | hl-helm gate plus a new unit suite for this chart; no service gates apply (no service code changes) |
| HL Budget V. Homelab-ready operations | ✅ | The app containers keep their contract. The exemption is on the platform's Postgres, not an app workload; deployment goes through hl-helm |
| HL Budget VI. Simplicity | ✅ | One value and one template block; no new component |
| HL Budget VII. API design | N/A | No API change |
| hl-helm I. Declarative IaC | ✅ | The live change is applied by `bootstrap/install.sh` from a committed chart; no `kubectl patch`/`edit` (a patch is only the documented fallback, reconciled in the same session — research R4) |
| hl-helm II. Chart per application | ✅ | Change stays inside the existing `postgres` chart |
| hl-helm III. Validate before apply | ✅ | `scripts/validate.sh`, then `install.sh --diff` reviewed by the owner immediately before applying (explicitly applied component) |
| hl-helm IV. Parity through values; safest defaults | ✅ | Chart default `false` (keeps AppArmor); the homelab turns it on in `bootstrap/values/postgres.yaml`; documented in the chart README (research R2) |
| hl-helm V. Secrets never plaintext | ✅ | No secret material |
| hl-helm VI. Idempotent, reversible | ✅ | Chart `version` 0.2.0 → 0.3.0; re-applying is a no-op; the PVC is untouched and has `resource-policy: keep`; turning the value off reverts to operator defaults |
| hl-helm VII. Homelab simplicity | ✅ | No new operator; reduced hardening justified in the chart README |

Result: **PASS**, no violations. Complexity Tracking is empty.

**Post-design re-check (after Phase 1)**: still PASS. The design adds no component. The only
trade-off (AppArmor off for Postgres) is recorded in research R1 and in the chart README, as
hl-helm VII requires.

## Project Structure

### Documentation (this feature)

```text
specs/012-postgres-apparmor-fix/
├── plan.md              # This file
├── research.md          # Phase 0: decisions R1–R6
├── data-model.md        # Phase 1: the chart value and the resulting pod security settings
├── quickstart.md        # Phase 1: apply and verify on the live cluster
├── contracts/
│   └── postgres-chart-values.md   # Phase 1: the new chart value's contract
├── checklists/
│   └── requirements.md
└── tasks.md             # Phase 2 (/speckit-tasks)
```

### Source Code (hl-helm repository)

```text
hl-helm/
├── charts/infrastructure/postgres/
│   ├── Chart.yaml                    # version 0.2.0 → 0.3.0
│   ├── values.yaml                   # cluster.appArmorUnconfined: false, with comment
│   ├── values.schema.json            # boolean, optional
│   ├── templates/cluster.yaml        # podSecurityContext block when the value is true
│   ├── ci/apparmor-unconfined-values.yaml   # new fixture: renders + schema-validates the "on" case
│   ├── tests/cluster_test.yaml       # new helm-unittest suite (on, off, protections kept)
│   ├── README.md.gotmpl              # "AppArmor exemption" section
│   └── README.md                     # regenerated by helm-docs
├── bootstrap/values/postgres.yaml    # cluster.appArmorUnconfined: true (the homelab)
├── scripts/validate.sh               # also runs helm unittest for charts that ship a tests/ dir
└── docs/bootstrap.md                 # "After a node restart": check Postgres and its apps; reading the AppArmor denial
```

No changes in `hl-budget-backend` or `hl-budget-frontend`. In this repository: the spec folder
and, after deployment, the README Features row.

**Structure Decision**: all code changes are in hl-helm on a branch named
`012-postgres-apparmor-fix` (matching this repository's branch), merged by PR after the live
apply succeeds. The apply is made from that committed branch, which hl-helm's `install.sh`
allows (it requires a clean tree, not `main`).

## Complexity Tracking

No violations to justify.
