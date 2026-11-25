# Data Model: Postgres AppArmor Fix

No application data model changes. HL Budget's schema, records and migrations are untouched. The
"model" here is the configuration that decides how the database pod runs.

## Chart value: `cluster.appArmorUnconfined`

| Attribute | Value |
|---|---|
| Chart | `hl-helm/charts/infrastructure/postgres` |
| Type | boolean, optional |
| Chart default (`values.yaml`) | `false`: the operator's own defaults, AppArmor on |
| Homelab (`bootstrap/values/postgres.yaml`) | `true` |
| Effect when `true` | the `Cluster` gets `spec.podSecurityContext` (below) |
| Effect when `false` | no `podSecurityContext` is rendered; the operator applies its defaults |

Validation: `values.schema.json` accepts only a boolean, so anything else fails at render time.

## Resulting pod security settings (value `true`)

Pod level, from the `Cluster`'s `podSecurityContext`:

| Setting | Value | Changed? |
|---|---|---|
| runAsNonRoot | true | same as today |
| runAsUser | 26 | same |
| runAsGroup | 26 | same |
| fsGroup | 26 | same |
| seccompProfile | RuntimeDefault | same |
| appArmorProfile | **Unconfined** | **new** |

Container level, the operator's default `securityContext`, not overridden:

| Setting | Value |
|---|---|
| allowPrivilegeEscalation | false |
| capabilities | drop ALL |
| privileged | false |
| readOnlyRootFilesystem | true |
| runAsNonRoot | true |
| seccompProfile | RuntimeDefault |

## State transitions of the database pod

```text
CrashLoopBackOff (recovery → signal denied → exit)
   │  helm upgrade sets podSecurityContext
   ▼
Pod recreated on the same volume → recovery (redo) → end-of-recovery checkpoint succeeds
   ▼
Ready (primary) ──► the backend's next restart connects → backend Ready → API 200
```

Reverse: setting the value to `false` and re-applying recreates the pod with the operator's
defaults (AppArmor on). On the affected kernels, that only works until the next start that needs
recovery.
