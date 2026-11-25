---

description: "Task list for 012-postgres-apparmor-fix"
---

# Tasks: Postgres AppArmor Fix

**Input**: Design documents from `specs/012-postgres-apparmor-fix/`

**Prerequisites**: plan.md, spec.md, research.md, data-model.md, contracts/postgres-chart-values.md, quickstart.md

**Tests**: Included. HL Budget constitution IV says tests ship with the change, and hl-helm
constitution III requires the validation gate before any apply. Plan R6 defines the suite.

**Organization**: Grouped by user story. Almost every change is in the **hl-helm** repository
(`~/Documents/Projects/hl-helm`, branch `012-postgres-apparmor-fix`). Paths starting with
`hl-helm/` are in that repository; other paths are in this one. The live cluster is reached
with `KUBECONFIG=~/.kube/homelab.yaml` through the SSH tunnel
`ssh -N -L 6443:127.0.0.1:6443 homelab`.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Can run in parallel (different files, no dependencies)
- **[Story]**: Which user story this task belongs to (US1, US2, US3)

---

## Phase 1: Setup

**Purpose**: Get the hl-helm working copy and the cluster connection ready

- [X] T001 In `hl-helm`, confirm a clean tree (`git status --short` empty), `git fetch origin`, and create branch `012-postgres-apparmor-fix` from `origin/main` (currently `d869256`); the local checkout is on `config/hl-budget-backend-money-timezone`, so do not reuse it
- [X] T002 [P] Confirm tools: `helm version` (4.1.x), `helm plugin list` shows `diff` and `unittest`, `kubeconform` and `helm-docs` on PATH, `python3 -c 'import yaml'`
- [X] T003 [P] Confirm cluster access and record the starting state: `kubectl -n postgres get pods,pvc` (`postgres-1` CrashLoopBackOff; note the PVC's AGE for the FR-007 check) and `kubectl -n hl-budget get pods`

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: Let the hl-helm gate run unit tests for an infrastructure chart (plan R6). Story
work needs this to pass the gate.

- [X] T004 Extend `hl-helm/scripts/validate.sh` "== unit tests" step: after the existing `helm unittest -f '*_test.yaml' tests/unit`, loop over `charts/*/*/` and, for each chart dir that has a `tests/` directory, run `helm unittest -f 'tests/*_test.yaml' "$dir"`, setting `fail=1` on failure, and print `unit ok: $dir` / `unit FAILED: $dir` — already on hl-helm `main` as `scripts/chart-tests.sh` (commit 36b5e1b), wired into `validate.sh` with self-test case 9; nothing to add
- [X] T005 Make sure chart-local test files are not packaged into the chart: add `tests/` to `hl-helm/charts/infrastructure/postgres/.helmignore` (create the file if it doesn't exist; check whether other charts have one and copy its header) — skipped by convention: the charts that already ship `tests/` (tailscale, ingress-tls) have no `.helmignore`
- [X] T006 Run `hl-helm/scripts/validate.sh` and confirm `GATE PASS` with no chart tests present yet (the loop finds none)

**Checkpoint**: the gate runs chart-local unit tests

---

## Phase 3: User Story 1 — HL Budget works again (Priority: P1) 🎯 MVP

**Goal**: Postgres finishes crash recovery on kernel 7.0.0-44, the backend becomes ready, and
the Spendings screen loads with all data intact.

**Independent Test**: after the apply, sign in at https://budget.valyalkin.com and open
Spendings, Income and Overview for October and November 2026 (quickstart §4–5).

### Tests for User Story 1 (write first; they fail until T011–T013)

- [X] T007 [US1] Create `hl-helm/charts/infrastructure/postgres/tests/cluster_test.yaml` (helm-unittest; `suite: cluster`, `templates: [templates/cluster.yaml]`, `set: {databases: []}` if the chart needs it to render) with these cases:
  - **(a) value on.** `set: {cluster.appArmorUnconfined: true}`. Assert on `kind: Cluster`: `spec.podSecurityContext` equals exactly `{runAsNonRoot: true, runAsUser: 26, runAsGroup: 26, fsGroup: 26, seccompProfile: {type: RuntimeDefault}, appArmorProfile: {type: Unconfined}}`, and `spec.securityContext` does not exist.
  - **(b) value off.** `set: {cluster.appArmorUnconfined: false}`. Assert that `spec.podSecurityContext` does not exist.
  - **(c) value unset.** Defaults; same assertion as (b).
  - **(d) schema.** `set: {cluster.appArmorUnconfined: "yes"}`. Assert `failedTemplate` (schema rejects a non-boolean).
- [X] T008 [US1] Run `helm unittest -f 'tests/*_test.yaml' hl-helm/charts/infrastructure/postgres` and confirm cases (a) and (d) FAIL (red) before implementation

### Implementation for User Story 1

- [X] T009 [P] [US1] Add to `hl-helm/charts/infrastructure/postgres/values.yaml` under `cluster:` the key `appArmorUnconfined: false`, with the comment from `contracts/postgres-chart-values.md`. The comment names the kernel regression (Ubuntu ≥ 7.0.0-43, AppArmor stacking, "could not signal for checkpoint: Permission denied"), hl-budget-spec 012, and the check for turning it off: `journalctl -k | grep 'DENIED.*signal.*comm="postgres"'` stays empty on a new kernel
- [X] T010 [P] [US1] Add `"appArmorUnconfined": {"type": "boolean"}` to `properties` of `cluster` in `hl-helm/charts/infrastructure/postgres/values.schema.json` (optional: do not add it to `required`)
- [X] T011 [US1] In `hl-helm/charts/infrastructure/postgres/templates/cluster.yaml`, after the `resources:` block and before `{{- with .Values.databases }}`, add `{{- if .Values.cluster.appArmorUnconfined }}` rendering `podSecurityContext:` with exactly the six fields from `contracts/postgres-chart-values.md` (`runAsNonRoot: true`, `runAsUser: 26`, `runAsGroup: 26`, `fsGroup: 26`, `seccompProfile: {type: RuntimeDefault}`, `appArmorProfile: {type: Unconfined}`). Add a one-line comment pointing at the values key. Do NOT render `securityContext`
- [X] T012 [US1] Bump `version: 0.2.0` → `0.3.0` in `hl-helm/charts/infrastructure/postgres/Chart.yaml` (`appVersion` stays `"17.7"`)
- [X] T013 [US1] Set `cluster: {appArmorUnconfined: true}` in `hl-helm/bootstrap/values/postgres.yaml`, with a two-line comment: why (homelab kernel ≥ 7.0.0-43, hl-budget-spec 012) and that the chart default stays `false`
- [X] T014 [P] [US1] Add fixture `hl-helm/charts/infrastructure/postgres/ci/apparmor-unconfined-values.yaml` with `databases: [{name: todo}]` and `cluster: {appArmorUnconfined: true}`, so `validate.sh` renders and kubeconform-checks the "on" case
- [X] T015 [US1] Add an "AppArmor exemption" section to `hl-helm/charts/infrastructure/postgres/README.md.gotmpl`. Cover: what the value does; why the homelab turns it on (the regression, with the denial line as an example); what it keeps (the six pod-level fields plus the container defaults: drop ALL, read-only root fs, no privilege escalation); how to check whether it's still needed and how to turn it off. This is the justification hl-helm constitution VII requires. Then run `helm-docs` in `hl-helm` to regenerate `charts/infrastructure/postgres/README.md`
- [X] T016 [US1] Run `helm unittest -f 'tests/*_test.yaml' hl-helm/charts/infrastructure/postgres` (all four cases green), then `hl-helm/scripts/validate.sh` (`GATE PASS`) and `hl-helm/scripts/validate.sh --self-test` (every injected violation still caught, since T004 touched the gate)
- [X] T017 [US1] Commit on `hl-helm` branch `012-postgres-apparmor-fix` (`fix(postgres): run Postgres pods AppArmor-unconfined on kernels >= 7.0.0-43`, Conventional Commits, with the Co-Authored-By trailer) and push; tree must be clean for `install.sh`
- [X] T018 [US1] **Owner step**: unseal Vault with `hl-helm/bootstrap/vault-init.sh --keys-file ~/.hl-helm/vault-init.json`; confirm `kubectl -n vault exec vault-0 -- vault status` shows `Sealed false`. Do NOT run `bootstrap/install.sh` without `--only` (it would re-provision; see the Vault-has-no-backup note)
- [X] T019 [US1] Run `hl-helm/bootstrap/install.sh --only postgres --diff` and show the owner the full diff. It must contain only `spec.podSecurityContext` on `Cluster/postgres` and the chart label `postgres-0.2.0` → `postgres-0.3.0` (helm-diff may also list the provision hook Job). Anything touching storage, the PVC or databases is a STOP. **Owner approves before T020**
- [X] T020 [US1] Apply with `hl-helm/bootstrap/install.sh --only postgres`. If the provision hook fails because Vault is unreachable, stop; don't use the `kubectl patch` fallback (research R4) without the owner's go-ahead
- [X] T021 [US1] Verify recovery (quickstart §4):
  - `kubectl -n postgres get pods -w` until `postgres-1` is `1/1 Running`.
  - The logs show `redo done` then `database system is ready to accept connections`, with no `could not signal for checkpoint`.
  - `kubectl -n postgres get pvc postgres-1` AGE is unchanged from T003 (FR-007).
  - If recovery fails for another reason, STOP. Never delete the PVC or the Cluster.
- [X] T022 [US1] Verify the pod's protections (SC-006) with `kubectl -n postgres get pod postgres-1 -o jsonpath='{.spec.securityContext}{"\n"}{.spec.containers[0].securityContext}'`. Pod level: `appArmorProfile Unconfined`, `runAsUser 26`, `runAsGroup 26`, `fsGroup 26`, `runAsNonRoot true`, seccomp `RuntimeDefault`. Container level: `drop ALL`, `readOnlyRootFilesystem true`, `allowPrivilegeEscalation false`
- [X] T023 [US1] Bring the backend back (quickstart §5): `kubectl -n hl-budget rollout restart deploy/hl-budget-backend`, wait for `1/1 Running`, and check that `curl -s -o /dev/null -w '%{http_code}' https://budget.valyalkin.com/api/v1/me` returns `401` (not `503`). Also confirm the VaultStaticSecrets in `hl-budget` and `postgres` report no new `VaultClientConfigError` events
- [X] T024 [US1] **Owner step**: sign in at https://budget.valyalkin.com and open Spendings, Income and Overview for October and November 2026. Months show entries, totals match pre-outage values (SC-001, SC-002), and there's no "We couldn't load your spendings"

**Checkpoint**: HL Budget is back; the outage is over (MVP)

---

## Phase 4: User Story 2 — The database survives the next unclean restart (Priority: P2)

**Goal**: an unclean stop of Postgres or the node no longer takes HL Budget down.

**Independent Test**: force-kill the database pod and confirm it is ready again within 2
minutes, with no signal denials (quickstart §6).

- [X] T025 [US2] Note the time, then run `kubectl -n postgres delete pod postgres-1 --grace-period=0 --force`; watch `kubectl -n postgres get pods -w` until `1/1 Running` and record the time taken (must be ≤ 2 minutes, SC-003)
- [X] T026 [US2] Confirm the restart went through recovery without denials:
  - `kubectl -n postgres logs postgres-1 -c postgres` shows recovery (`redo` or `database system was interrupted`) and no `could not signal`.
  - `ssh homelab 'journalctl -k --since "-10min" | grep -c "DENIED.*signal.*comm=\"postgres\""'` prints `0`.
- [X] T027 [US2] Confirm HL Budget is unaffected after T025: backend pod `1/1 Running` (restart it if it lost its connection pool), `/api/v1/me` → `401`, and the Spendings screen loads
- [X] T028 [US2] Record that a full node reboot (spec US2 scenario 2, SC-004) is NOT forced in this feature. It was observed on 2026-11-25 (reboot into 7.0.0-44), and it would re-seal Vault. Note in `specs/012-postgres-apparmor-fix/tasks.md` Notes that the next real reboot is the check: Postgres and the backend must come back without manual steps other than the Vault unseal

**Checkpoint**: the latent failure is gone for unclean restarts

---

## Phase 5: User Story 3 — The fix is recorded where the platform is defined (Priority: P3)

**Goal**: the chart, the docs and the live cluster agree, and an operator can see why the
exemption exists and how to undo it.

**Independent Test**: `install.sh --only postgres --diff` shows no differences, and the chart's
rendered defaults keep AppArmor (quickstart §7).

- [X] T029 [US3] Run `hl-helm/bootstrap/install.sh --only postgres --diff` again and confirm it shows no differences (SC-005; idempotence, hl-helm constitution VI)
- [X] T030 [P] [US3] Add a paragraph to `hl-helm/docs/bootstrap.md` § "After a node restart" (FR-009). After unsealing Vault, check `kubectl -n postgres get pods` and the apps that use it. If `postgres-1` crash-loops with `could not signal for checkpoint: Permission denied`, check `journalctl -k | grep 'DENIED.*signal.*comm="postgres"'` on the node and confirm `cluster.appArmorUnconfined: true` in `bootstrap/values/postgres.yaml`. Link the chart README section from T015
- [X] T031 [P] [US3] Add a row to `hl-helm/docs/validation-results.md` (match its existing table format) recording the 2026-11-25 incident check: the forced unclean restart from T025–T026, its recovery time and zero denials
- [X] T032 [US3] Run `hl-helm/scripts/validate.sh` (`GATE PASS`), commit the doc changes on `hl-helm` branch `012-postgres-apparmor-fix` (`docs: ...`), push, and open the hl-helm PR with `gh pr create`. Include in the body: a summary, the link to hl-budget-spec 012, the `--diff` output from T019 (hl-helm constitution III), verification results from T021–T027, and the 🤖 Generated with Claude Code line
- [ ] T033 [US3] **Owner step**: review and merge the hl-helm PR. Then confirm `git -C hl-helm log origin/main` contains the change, so the next `install.sh` from `main` keeps the fix

**Checkpoint**: repository and cluster agree; the next upgrade keeps the fix

---

## Phase 6: Polish & Cross-Cutting Concerns

- [X] T034 [P] Update `specs/012-postgres-apparmor-fix/spec.md` **Status** line to "Implemented and deployed <date>", naming the hl-helm PR number, and add an "Implementation notes" section with anything learned during T018–T027 (as in specs 010/011)
- [X] T035 [P] Add the 012 row to the Features table in `README.md`, following the existing rows ("Implemented <date>: Postgres pods run AppArmor-unconfined on kernels ≥ 7.0.0-43 so crash recovery works; HL Budget restored after the 2026-11-23 outage; platform only")
- [X] T036 Commit this repository's changes on branch `012-postgres-apparmor-fix` (`spec(012): ...` / `docs: ...`), push, open the PR with `gh pr create`, and mark all tasks in this file done
- [ ] T037 [P] Record the follow-ups from spec "Out of scope" as GitHub issues in this repository, only if the owner wants them now: crash-loop alerting, planned reboots after kernel updates, Vault auto-unseal, filing the Ubuntu kernel/AppArmor bug. Ask first; don't open them unprompted — deferred by the owner on 2026-11-25: do not open the issues yet

---

## Dependencies & Execution Order

### Phase dependencies

- **Setup (1)** → **Foundational (2)** → **US1 (3)** → **US2 (4)** → **US3 (5)** → **Polish (6)**
- US2 depends on US1: the fix must be live before an unclean restart can be tested.
- US3's merge (T033) should follow US1/US2 verification, so the PR carries real results.
  T030/T031 (docs) can be written any time after T017.

### Within US1

- T007 → T008 (red) → T009, T010 [P] → T011 → T012, T013, T014 [P] → T015 → T016 (green +
  gate) → T017 (commit) → T018 (owner unseal) → T019 (diff, owner approval) → T020 (apply) →
  T021 → T022 → T023 → T024 (owner check)

### Owner gates (cannot be automated)

- T018 Vault unseal (holds the unseal key)
- T019 diff approval (hl-helm constitution III: a human reviews before apply)
- T024 visual check of data
- T033 PR merge

## Parallel Example: User Story 1

```text
# Different files, no dependencies between them:
T009  values.yaml key + comment
T010  values.schema.json property
T014  ci/apparmor-unconfined-values.yaml fixture
```

## Parallel Example: User Story 3

```text
T030  docs/bootstrap.md paragraph
T031  docs/validation-results.md row
```

## Implementation Strategy

### MVP first (User Story 1)

1. Phases 1–2: branch, tools, gate extension.
2. Phase 3 through T024: chart change, gate green, owner unseals and approves the diff, apply,
   HL Budget back.
3. **Stop and validate**: the outage is over at this checkpoint. Everything after hardens and
   records the fix.

### Incremental delivery

1. US1: restore service (ends the outage)
2. US2: prove an unclean restart now recovers
3. US3: docs, PR merged, chart = cluster
4. Polish: spec status, README row, this repository's PR

## Notes

- Never delete or recreate `pvc/postgres-1` or `cluster/postgres`. `helm.sh/resource-policy:
  keep` protects the Cluster from a release delete, but not from a manual delete.
- Never run `bootstrap/install.sh` without `--only postgres` during this feature.
- `kubectl patch` is only the research R4 fallback, used with the owner's go-ahead and
  reconciled through `install.sh` in the same session.
- T028: a full node reboot was not forced. The next real reboot is the check for spec US2
  scenario 2 / SC-004: Postgres and the HL Budget backend must come back with no manual step
  other than the Vault unseal.
- Implementation (2026-11-25):
  - T004 was already on hl-helm `main` (`scripts/chart-tests.sh`).
  - At T020, CloudNativePG did not roll the crash-looping pod ("Cannot extract Pod status"). It
    needed `kubectl -n postgres delete pod postgres-1`, recreated on the same PVC. This is now in
    hl-helm `docs/bootstrap.md`.
  - At T018, the first unseal attempt hit the local k3d Vault because the shell's default kube
    context is `k3d-homelab-test`.
  - At T032, `gh pr create` failed with a GraphQL error, so the PR was opened through the REST
    API.

