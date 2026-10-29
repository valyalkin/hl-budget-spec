# Quickstart: Spending Screens (Web App)

**Feature**: `003-spending-screens` | **Plan**: [plan.md](./plan.md) |
**Contract**: [contracts/openapi.yaml](./contracts/openapi.yaml) |
**Data model**: [data-model.md](./data-model.md)

This guide proves the feature works. Each step names the spec item it validates.

## 0. Prerequisites

- Both worktrees (`hl-budget-backend/`, `hl-budget-frontend/`) on branch `003-spending-screens`
  from `origin/main` (research R-014).
- Docker, Java 21, Node 22, pnpm 10. Backend and frontend env files as in 001 and 002; this
  feature adds **no** environment variables.
- The Auth0 dev/test tenant's seeded user, exported as `E2E_USERNAME` / `E2E_PASSWORD`, and an API
  token for that user as `TOKEN` (001 quickstart).

## 1. Backend gates and contract

```sh
cd hl-budget-backend
./gradlew check
```

Expected: green, including the ITs in research R-013. `OpenApiExportIT` rewrites
`contracts/api.yaml`; `git diff` shows only what [contracts/openapi.yaml](./contracts/openapi.yaml)
describes: three new paths, the relaxed `SpendingRequest`, the new 422 on account delete.

## 2. Backend checks by hand

```sh
./gradlew bootRun --args='--spring.profiles.active=local'
API=http://localhost:8080/api/v1
auth=(-H "Authorization: Bearer $TOKEN" -H 'Content-Type: application/json')
```

| # | Request | Expect | Validates |
|---|---|---|---|
| 1 | `GET $API/monthly-totals` | 200, `year` = this year, 12 `months`, `today` = today in the app zone | R-002, FR-010, FR-012 |
| 2 | `POST $API/spendings` `{"date":"<today>","merchant":"Riverside Café","amount":{"amount":"6.75","currency":"CAD"},"categoryName":"coffee","accountName":"Visa"}` with `Idempotency-Key: k3` (no "Visa" account yet) | 201; category `Coffee` reused if it exists in any case, else created as `coffee`; account `Visa` created | FR-035, R-004 |
| 3 | Repeat #2 | 200 `Idempotent-Replayed`; `GET $API/accounts` has one `Visa` | FR-039 |
| 4 | `POST` with both `categoryId` and `categoryName` | 422, pointer `/categoryId` | R-004 |
| 5 | `POST` with tomorrow's date and `"accountName":"Brand new"` | 422 on `/date`; `GET $API/accounts` has no `Brand new` | R-004 (nothing created) |
| 6 | `GET $API/category-totals?from=<month start>&to=<month end>` | Only categories with spending, largest first, totals add up to the envelope total | FR-071, FR-072 |
| 7 | `GET $API/merchants` | `Riverside Café` with `lastCategory` and `lastAccount` from #2 | FR-057, FR-058 |
| 8 | Delete spendings until one account is left with none; `DELETE` it | 422, `details.reason: last-account` | FR-084, FR-086 |
| 9 | #1, #6, #7 with another user's token | Only that user's data | Constitution III |

## 3. Frontend gates

```sh
cp hl-budget-backend/contracts/api.yaml hl-budget-frontend/contracts/api.yaml
cd hl-budget-frontend
pnpm generate:api
pnpm typecheck && pnpm lint && pnpm test && pnpm build
```

Expected: all green; CI's "schema matches contract" check passes.

## 4. Frontend journey (local, backend running)

`pnpm dev`, then in a browser:

| # | Do | Expect | Validates |
|---|---|---|---|
| 1 | Open `/` signed out | Header with Sign in; "You're signed out"; no data. `/spendings` redirects to sign-in | US7, FR-002 |
| 2 | Sign in | Lands on `/spendings`, current month selected, future months greyed, next-year arrow disabled | US1, FR-010–FR-012 |
| 3 | Pick an empty past month | "No spendings recorded in [Month Year]."; card 0 | FR-022 |
| 4 | Click +, type a merchant used before, press Tab | Merchant autofilled; category and account filled from its last spending | US3 AS-10, FR-058 |
| 5 | Type `g` in category | Most-used `g…` category filled in with the rest selected; Backspace removes the rest | FR-050, FR-052 |
| 6 | Tab to amount, type `12.50`, press Enter | Saved once; row closes; table, footer, month line, both cards updated without reload | FR-038, SC-001, SC-003 |
| 7 | Add a row dated last month | View switches to last month | FR-038 |
| 8 | Leave merchant empty, press Enter on amount | "Add a merchant name.", focus in merchant | FR-055 |
| 9 | Hover a row, then Tab into a row | Edit and remove appear both ways | FR-031, FR-091 |
| 10 | Edit a row's amount and save; then remove it | Updated, then gone at once, totals follow | US4, FR-060 |
| 11 | By category: toggle Pie, change month | Pie kept; one slice per category with spending, no "Everything else" | US5, FR-070, FR-072 |
| 12 | User icon → Settings → Categories: add `coffee` when `Coffee` exists; add a new one; remove it; remove a used one | "“coffee” already exists."; added; removed; "X is used by N spendings…" | US6 |
| 13 | Accounts: remove a used account; remove the only unused last account | "X has N spendings…"; "You need at least one account." | FR-084 |
| 14 | Escape / backdrop click | Dialog closes; Spendings reflects changes | FR-080, FR-085 |
| 15 | Sign out, press Back | Signed-out screen; no spendings | US7 AS-2 |

## 5. End-to-end and accessibility

```sh
export E2E_USERNAME=… E2E_PASSWORD='…'
set -a; . ./.env.local; set +a
pnpm e2e
```

Expected: one passing spec per user story (SC-006) and the axe scans with no serious or
critical issues (SC-008). The suite seeds its own data through the UI and removes it afterwards.

## 6. Performance spot check (SC-005)

Backend `TotalsPerformanceIT` seeds 10,000 spendings and asserts each of `monthly-totals`,
`category-totals`, `merchants` and a month's spending list answer in under 200 ms. In the
browser, switching months on that data set shows the new month in under 1 s.

## 7. Deploy

1. Merge the backend PR; release workflow → hl-helm image PR → merge. The new reads are additive,
   so the running frontend is unaffected.
2. Merge the frontend PR; same path.
3. `homelab:hl-deploy-status` for both: Argo CD Healthy, pods Ready. Sign in on the live host
   and repeat §4 rows 2, 6 and 11.
