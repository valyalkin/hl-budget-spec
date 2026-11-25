# Contract: Postgres chart value `cluster.appArmorUnconfined`

Interface: the values of `hl-helm/charts/infrastructure/postgres` (chart `0.3.0`). Operators of
the chart set it in a values file; `bootstrap/install.sh` passes `bootstrap/values/postgres.yaml`.

## Input

```yaml
cluster:
  # Run the Postgres pods without an AppArmor profile. Needed on Ubuntu kernels >= 7.0.0-43, where
  # AppArmor stacking stops Postgres processes signalling each other and crash recovery fails with
  # "could not signal for checkpoint: Permission denied" (hl-budget-spec 012). Turn off once
  # `journalctl -k | grep 'DENIED.*signal.*comm="postgres"'` stays empty on a new kernel.
  appArmorUnconfined: false
```

| Key | Type | Required | Default | Allowed |
|---|---|---|---|---|
| `cluster.appArmorUnconfined` | boolean | no | `false` | `true`, `false` |

A non-boolean value MUST fail `helm template` and `helm lint` through `values.schema.json`.

## Output (rendered `Cluster`)

When `true`, the `postgresql.cnpg.io/v1` `Cluster` MUST contain exactly:

```yaml
spec:
  podSecurityContext:
    runAsNonRoot: true
    runAsUser: 26
    runAsGroup: 26
    fsGroup: 26
    seccompProfile:
      type: RuntimeDefault
    appArmorProfile:
      type: Unconfined
```

When `false` or unset, the `Cluster` MUST NOT contain `spec.podSecurityContext`.

In both cases the `Cluster` MUST NOT contain `spec.securityContext`, so the operator's
container defaults stay in force. Nothing else in the rendered output changes.

## Compatibility

- Requires CloudNativePG ≥ 1.26, which added `podSecurityContext`; the homelab runs 1.30.1.
  Requires Kubernetes ≥ 1.30 for `appArmorProfile`; the homelab runs 1.36.
- Backward compatible: existing values files without the key render exactly as chart 0.2.0 did.
- Chart `version` MUST advance from `0.2.0` to `0.3.0` (a new value is a minor change).
