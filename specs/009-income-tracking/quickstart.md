# Quickstart: validating feature 009

How to prove the feature works. Shapes are in [contracts/openapi-delta.yaml](./contracts/openapi-delta.yaml)
and [data-model.md](./data-model.md); decisions in [research.md](./research.md).

## 1. Prerequisites

- Docker running (Testcontainers), Node 22 + pnpm, Java 21.
- Branch `009-income-tracking` in `hl-budget-backend` and `hl-budget-frontend`, from `origin/main`
  (both include feature 008).

## 2. Gates

```sh
# hl-budget-backend
./gradlew check            # ktlint, detekt, unit + ITs (incl. the V7 migration IT), Kover floors

# hl-budget-frontend
pnpm generate:api          # after copying the backend's exported contracts/api.yaml
pnpm typecheck && pnpm lint && pnpm test
pnpm e2e                   # desktop and phone projects, axe scans; every pre-009 spec unchanged and green
```

## 3. Backend checks (local, `./gradlew bootRun`, token for a test user as in 003's quickstart)

1. **Defaults**: as a new user, `GET /api/v1/income-categories` → exactly Investments `#1d4ed8`
   and Salary `#1e5b45`, both `isDefault: true`, `incomeCount: 0`. As a user created before V7
   (has spending categories already) → the same two, once. Ten parallel first calls → still two.
2. **Separate lists**: `POST /api/v1/income-categories` "Other" → `201` (a spending category
   "Other" exists). "salary" → `409 duplicate-name`. Rename or delete Salary → `422
   default-category`. `GET /api/v1/categories` → still exactly the spending categories.
3. **Colours**: the first custom income category gets `#e11d48`, the next `#0891b2`; rename keeps
   the colour.
4. **Create income**: `POST /api/v1/incomes` with `categoryName: Salary`, an existing
   `accountName`, `Idempotency-Key: k1` → `201` with `source`, `category`, `account`. Repeat with
   `k1` → same entry, still one. `categoryName: Groceries` → `422`, `#/categoryName` "is not one of
   your income categories"; nothing created. Future date, amount `0`, blank source → `422` with the
   spendings' messages.
5. **Spending side unchanged**: `POST /api/v1/spendings` with `categoryName: Salary` → `422
   NotFound`. `GET /api/v1/monthly-totals` and `/category-totals` ignore income.
6. **Totals**: record October income (Salary 5000.00 on Debit card, Investments 120.25 and 200.25
   on Brokerage). `GET /api/v1/monthly-income-totals?year=2026` → October `5320.50`, `entryCount:
   3`. `/income-category-totals` and `/income-account-totals` for October → items largest first,
   each with its colour, totals adding up to `5320.50`. Empty month → `items: []`, total `0.00`.
   Range rules as category totals.
7. **Sources**: `GET /api/v1/income-sources` → "Acme Corp" with `lastCategory` Salary and
   `lastAccount` Debit card; merchants are not listed, and `/merchants` lists no source.
8. **Accounts**: `GET /api/v1/accounts` → each has `incomeCount`. Delete Brokerage (used only by
   income) → `409 in-use`, `details: { spendingCount: 0, incomeCount: 2 }`. Delete income category
   Investments' entries, then the account → `204` (if it is not the last one).
9. **Import**: `POST /api/v1/income-imports` with 2 valid rows → `201`, `incomeCount: 2`; with one
   row naming "Groceries" → `422` at `#/incomes/1/categoryName`, nothing saved; repeat a key →
   same receipt.
10. **Isolation**: a second user can't read, change, delete or reference the first user's incomes or
    income categories (`404` / `422 NotFound`); their totals never include them (`OwnershipIT`).
11. **Performance**: performance IT — 10,000 spendings and 1,000 incomes; each income endpoint for a
    month well under 1 s.

## 4. Frontend checks (`pnpm dev` against the local backend)

Desktop (wide window):

1. The header nav shows **Spendings** and **Income**; Spendings is selected on opening. Select
   September 2026, click Income → `/income?month=2026-09`, no full reload, September selected;
   click Spendings → September again. Reload → the current tab's URL stays; opening `/` → Spendings.
2. Income month line shows income totals only; the table columns read Date, Source, Category,
   Account, Amount; footer "3 entries"; empty month → "No income recorded in [Month Year]."
3. Click + → row with calendar open; type "ac" → "Acme Corp" autofills, Tab fills Salary and Debit
   card; Enter saves. The category dropdown lists only Salary, Investments and custom income
   categories. Type "Groceries" → "Not one of your income categories", no save.
4. Hover a row → edit and remove; remove deletes at once; every figure updates without a reload.
5. By category and By account cards: Salary green, Investments blue; hover labels; Table / Pie
   choices on Income are independent of those on Spendings and survive a month change and reload.
6. Import CSV on Income → title "Import income", sample file has Salary and Investments rows; a row
   with "Groceries" is flagged; Add saves all rows as income; Spendings unchanged.
7. Settings → **Spending categories**, **Income categories**, **Accounts**. Add income category
   "Freelance" → it shows a swatch and appears only in the Income dropdown. Remove it while used →
   "Freelance is used by 1 entry, so it can’t be removed." An account used by income →
   "3 spendings · 1 income entry"; removing it shows both counts.
8. Switching tabs with an open add row discards it; no other change on Spendings (existing e2e green).

Phone (390 × 844, and 360 wide):

9. Header switch "Spendings | Income", each option at least 44px, no sideways scroll.
10. Income: Month and Year dropdowns, income total, switch "Income | By category | By account",
    list, floating "Add income" → full-screen form with Source; Edit income; delete asks in a sheet.
11. Back gesture with a sheet open closes the sheet; with nothing open it does not switch tabs.
12. Empty month → "No income in [Month Year]."; By category / By account with the top-6 donut.

## 5. After deploy

- `homelab:hl-deploy-status` for both services (Argo CD Synced/Healthy, backend on schema V7).
- At https://budget.valyalkin.com: Income tab shows Salary and Investments for the existing user;
  record one income on desktop and phone; Spendings figures unchanged.
- Add the 009 row to the README Features table.
