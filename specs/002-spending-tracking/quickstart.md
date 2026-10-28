# Quickstart: Spending Tracking (Backend)

**Feature**: `002-spending-tracking` | **Plan**: [plan.md](./plan.md) |
**Contract**: [contracts/openapi.yaml](./contracts/openapi.yaml) |
**Data model**: [data-model.md](./data-model.md)

This guide proves the feature works. Each step names the spec item it validates.

## 0. Prerequisites

- Docker running; Java 21; the `hl-budget-backend` worktree at `hl-budget-spec/hl-budget-backend/`
  on branch `002-spending-tracking` (research R-001).
- `.env.local` from `.env.example`, now including `HLBUDGET_MONEY_CURRENCY=CAD` and
  `HLBUDGET_TIMEZONE=America/Toronto` (R-006). The local profile sets them too.
- An access token for the HL Budget API audience (as in 001's quickstart), exported as `TOKEN`.
  A second user's token as `TOKEN_B` for the ownership checks.

## 1. Gates

```sh
cd hl-budget-backend
./gradlew check      # ktlint, detekt, unit + Testcontainers ITs (incl. the 10,000-row SC-004 check), Kover gates
```

Expected: green. `OpenApiExportIT` rewrites `contracts/api.yaml`, and `git diff` shows only the
additions described in [contracts/openapi.yaml](./contracts/openapi.yaml) plus 422 entries
(R-009). The ITs listed in research R-013 are the automated form of every check below.

## 2. Run locally

```sh
./gradlew bootRun --args='--spring.profiles.active=local'
API=http://localhost:8080/api/v1
auth=(-H "Authorization: Bearer $TOKEN" -H 'Content-Type: application/json')
```

Startup applies `V2` and `V3`. Startup must **fail** naming the property if
`HLBUDGET_MONEY_CURRENCY` or `HLBUDGET_TIMEZONE` is unset or invalid (try `XYZ`).

## 3. Manual journey

| # | Request | Expect | Validates |
|---|---|---|---|
| 1 | `curl "${auth[@]}" $API/categories` | 200, exactly 17 items, all `isDefault: true`, `spendingCount: 0` | US1, FR-010 |
| 2 | Repeat #1 | Same 17, no duplicates | FR-011 |
| 3 | `curl "${auth[@]}" -X POST $API/accounts -d '{"name":"Amex"}'` | 201, `Location`, `spendingCount: 0` | US2, SC-001 (1 of 2) |
| 4 | Same with `{"name":" amex "}` | 409, `details.reason: duplicate-name` | FR-022 |
| 5 | `POST $API/spendings` with `Idempotency-Key: k1`, body from the contract example (use ids from #1 and #3) | 201 with category and account names and `amount.amount: "84.37"` | US3, SC-001 (2 of 2) |
| 6 | Repeat #5 exactly | 200, `Idempotent-Replayed: true`, same `id` | FR-036, SC-007 |
| 7 | Repeat #5 with `k1` and a different merchant | 422 `idempotency-key-reused` | R-007 |
| 8 | `POST $API/spendings` with tomorrow's date, `"amount":"0"`, `"currency":"EUR"` | 422 with three `errors`: `#/date` NotFuture, `#/amount/amount` Positive, `#/amount/currency` UnsupportedCurrency | FR-005, FR-030, FR-032 |
| 9 | `POST` with `"amount": 84.37` (a JSON number) | 422 `TypeMismatch` at `#/amount/amount` | R-006 |
| 10 | `GET $API/spendings?from=2026-10-01&to=2026-10-31&sort=amount,desc&size=5` | 200 page, `totalItems` correct | US4 |
| 11 | `GET $API/spendings?size=101` and `?from=2026-11-01&to=2026-10-01` and `?sort=merchant,asc` | 422 naming `size` / `from` / `sort` | Edge cases |
| 12 | `GET $API/accounts` | Amex with `spendingCount: 1` | FR-026 |
| 13 | `DELETE $API/accounts/{amex}` | 409 `in-use`, `spendingCount: 1` | FR-025 |
| 14 | `PUT $API/categories/{groceries}` `{"name":"Food"}`; `DELETE` the same | 422 `default-category` both times | FR-015 |
| 15 | `POST $API/categories` `{"name":"Coffee"}`; `PUT` the spending to Coffee; `PUT` Coffee to `Cafés` | `GET` the spending shows `Cafés` | US5, FR-034 |
| 16 | `DELETE` the spending, then `DELETE` Amex | 204, 204; `GET` the spending is 404 | US3, FR-025 |
| 17 | Every request above with `TOKEN_B` against user A's ids | 404 (path) or 422 `NotFound` (reference); A unchanged | FR-002, SC-003 |
| 18 | `PUT $API/users/me` | Unchanged from 001 (200/201, same body) | FR-041 |

## 4. Frontend types

```sh
cp hl-budget-backend/contracts/api.yaml hl-budget-frontend/contracts/api.yaml
cd hl-budget-frontend && pnpm generate:api && pnpm typecheck && pnpm lint && pnpm test
```

Expected: `app/lib/api-schema.d.ts` gains the new operations; all gates green; no UI change.

## 5. Deploy

1. `homelab:hl-service-config`: the hl-helm PR adds `HLBUDGET_MONEY_CURRENCY` and
   `HLBUDGET_TIMEZONE` to the backend env in `charts/apps/hl-budget/values.yaml`. **Merge it before
   the new image.** The running 001 image ignores the new variables, but the new image refuses to
   start without them.
2. Merge the backend PR; the release workflow opens the image-bump PR in hl-helm; merge it.
3. `homelab:hl-deploy-status`: Argo CD Healthy, pod Ready, the log shows Flyway `V2` and `V3`
   applied.
4. From the web app's network, sign in, then repeat journey rows 1, 3 and 5 against the deployed
   host with a real token.
