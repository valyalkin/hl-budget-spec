# Quickstart: validating feature 007

How to prove the feature works. Shapes are in [contracts/openapi-delta.yaml](./contracts/openapi-delta.yaml)
and [data-model.md](./data-model.md); decisions in [research.md](./research.md).

## 1. Prerequisites

- Docker running (Testcontainers), Node 22 + pnpm, Java 21.
- Branch `007-fix-pie-csv-categories` in `hl-budget-backend` and `hl-budget-frontend`.

## 2. Gates

```sh
# hl-budget-backend
./gradlew check            # ktlint, detekt, unit + ITs (incl. the V5 migration IT), Kover floors

# hl-budget-frontend
pnpm generate:api          # after copying the backend's exported contracts/api.yaml
pnpm typecheck && pnpm lint && pnpm test
pnpm e2e                   # desktop and phone projects, axe scans
```

## 3. Backend checks (local, `./gradlew bootRun`, token for a test user as in 003's quickstart)

1. **Default colours**: `GET /api/v1/categories` → 17 defaults, each with `color`; Rent is
   `#1e5b45`, Taxi `#8a6d00`; 17 different values. Repeat with a second user → same colours.
2. **New category colour**: `POST /api/v1/categories {"name":"Coffee"}` → `201` with `color`
   `#e11d48` (first spare). A second → `#0891b2`. Rename Coffee → colour unchanged.
3. **Totals carry colours**: record a Coffee spending; `GET /api/v1/category-totals?from=…&to=…`
   → the Coffee item has `color: "#e11d48"`.
4. **No creation by name**: `POST /api/v1/spendings` with `"categoryName":"Groceriess"` → `422`,
   `errors[0].pointer = "#/categoryName"`, `code = "NotFound"`; `GET /api/v1/categories` count
   unchanged. With `"categoryName":"  groceries "` → `201`, category "Groceries".
5. **Same for accounts**: `"accountName":"Visa Gold"` (not existing) → `422` on `#/accountName`.
6. **Import refuses unknown names**: `POST /api/v1/spending-imports` with 6 rows, row 2 an unknown
   category and row 5 an unknown account → `422` with exactly those two pointers; no spendings,
   categories or accounts created.
7. **Back-fill**: covered by the migration IT (a user on V4 with defaults and 10 custom
   categories → defaults by name, customs 18…24 then 18, 19, 20 in creation order per R-005).

## 4. Frontend checks (`pnpm dev` against the local backend)

1. **Colours (US1)**: a month with spending in 12+ categories. Pie view: every slice filled; each
   legend swatch matches; Table view shows the same swatches. Change month: Groceries keeps its
   colour. Phone width (390px): same.
2. **Label (US2)**: hover each slice → centre shows its name and "amount · share" at once, others
   dim; leave → "Total" returns. Tab through slices → same label, screen reader announces it. Phone:
   tap a slice → label; tap outside → gone.
3. **Manual entry (US3)**: entry row, category "Groceriess" → highlighted "Not one of your
   categories", Enter does not save; "groceries" → shows "Groceries", saves. Account "Visa Gold" →
   "Not one of your accounts". Phone Add screen: same. Settings counts unchanged.
4. **No accounts**: a fresh user with no accounts → the entry row and phone Add screen show "Add an
   account in Settings first." with a working link to Settings › Accounts.
5. **Import (US4)**: import a file with "groceries", "Groceriess", an unknown account and an empty
   category → first valid as "Groceries", the other three highlighted, "Add spendings" greyed out.
   Fix them, add → spendings appear; Settings counts unchanged. Phone Fix row: same messages.
6. **Removed meanwhile**: open the review with a valid custom category, delete that (unused)
   category in Settings in another tab, press Add → refused, review stays, the field shows "Not one
   of your categories".

## 5. After deploy

`homelab:hl-deploy-status` for both services; repeat §4.1–4.3 on https://budget.valyalkin.com on
a desktop and a phone; then update the README Features table.
