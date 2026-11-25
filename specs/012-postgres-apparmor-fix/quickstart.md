# Quickstart: apply and verify the Postgres AppArmor fix

Everything runs from the owner's Mac against the homelab.

## Prerequisites

- SSH tunnel to the cluster: `ssh -N -L 6443:127.0.0.1:6443 homelab`
- `export KUBECONFIG=~/.kube/homelab.yaml`
- hl-helm checked out on branch `012-postgres-apparmor-fix` with the change committed, and a clean
  tree
- Tools: `helm` 4.1.1 with the `diff` and `unittest` plugins, `kubeconform`, `helm-docs`,
  `python3` + PyYAML
- The Vault unseal key file `~/.hl-helm/vault-init.json`

## 1. Gate (before touching the cluster)

```sh
cd hl-helm
scripts/validate.sh              # expect: GATE PASS
scripts/validate.sh --self-test  # the gate was touched; expect every injected violation caught
helm template ci charts/infrastructure/postgres -f bootstrap/values/postgres.yaml \
  | grep -A9 'podSecurityContext:'
```

Expected: the six fields from [contracts/postgres-chart-values.md](contracts/postgres-chart-values.md).

## 2. Unseal Vault

The provision Job runs on `pre-upgrade` and needs Vault.

```sh
bootstrap/vault-init.sh --keys-file ~/.hl-helm/vault-init.json
kubectl -n vault exec vault-0 -- vault status | grep Sealed    # Sealed  false
```

## 3. Review the diff, then apply

```sh
bootstrap/install.sh --only postgres --diff
```

Expected diff: only `spec.podSecurityContext` added to `Cluster/postgres`, plus the chart label
moving to `postgres-0.3.0`. Anything else is a reason to stop. Paste the diff into the hl-helm
PR description.

```sh
bootstrap/install.sh --only postgres
```

## 4. Verify the database recovers (US1, FR-001, FR-007)

```sh
kubectl -n postgres get pods -w          # postgres-1 recreated → 1/1 Running
kubectl -n postgres logs postgres-1 -c postgres | grep -oE '"message":"(redo done|database system is ready)[^"]*"'
kubectl -n postgres get pvc postgres-1   # same volume, not recreated (AGE unchanged)
```

Expected: `redo done …` followed by `database system is ready to accept connections`, and no
`could not signal for checkpoint`.

Check the running pod's protections (SC-006):

```sh
kubectl -n postgres get pod postgres-1 -o jsonpath='{.spec.securityContext}{"\n"}{.spec.containers[0].securityContext}{"\n"}'
```

Expected: pod-level `appArmorProfile {type: Unconfined}`, `runAsUser 26`, `runAsGroup 26`,
`fsGroup 26`, `runAsNonRoot true`, seccomp `RuntimeDefault`. Container-level: `drop ALL`,
`readOnlyRootFilesystem true`, `allowPrivilegeEscalation false`.

## 5. Verify HL Budget (US1, FR-008, SC-001, SC-002)

```sh
kubectl -n hl-budget rollout restart deploy/hl-budget-backend   # optional: skip the crash-loop back-off
kubectl -n hl-budget get pods                                    # both 1/1 Running
curl -s -o /dev/null -w '%{http_code}\n' https://budget.valyalkin.com/api/v1/me   # 401 (backend up, no token), not 503
```

In a browser, sign in at https://budget.valyalkin.com and open Spendings, Income and Overview for
October and November 2026. The months show their entries, and the totals match what the owner
remembers or has exported from before 2026-11-23.

## 6. Verify the unclean-restart case (US2, SC-003)

```sh
kubectl -n postgres delete pod postgres-1 --grace-period=0 --force
kubectl -n postgres get pods -w          # back to 1/1 within 2 minutes
ssh homelab 'journalctl -k --since "-5min" | grep -c "DENIED.*signal.*comm=\"postgres\""'   # 0
```

A forced delete kills Postgres without a clean shutdown, so the restart goes through crash
recovery. That's the path that failed before the fix.

## 7. Verify the chart matches the cluster (US3, SC-005)

```sh
bootstrap/install.sh --only postgres --diff    # expect: no differences
```

Then merge the hl-helm PR and add the feature's row to this repository's README Features table.
