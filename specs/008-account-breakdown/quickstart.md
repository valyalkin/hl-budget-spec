# Quickstart: validating feature 008

How to prove the feature works. Shapes are in [contracts/openapi-delta.yaml](./contracts/openapi-delta.yaml)
and [data-model.md](./data-model.md); decisions in [research.md](./research.md).

## 1. Prerequisites

- Docker running (Testcontainers), Node 22 + pnpm, Java 21.
- Branch `008-account-breakdown` in `hl-budget-backend` and `hl-budget-frontend`, from
  `origin/main`.

## 2. Gates

```sh
# hl-budget-backend
./gradlew check            # ktlint, detekt, unit + ITs (incl. the V6 migration IT), Kover floors

# hl-budget-frontend
pnpm generate:api          # after copying the backend's exported contracts/api.yaml
pnpm typecheck && pnpm lint && pnpm test
pnpm e2e                   # desktop and phone projects, axe scans
```

## 3. Backend checks (local, `./gradlew bootRun`, token for a test user as in 003's quickstart)

1. **New account colours**: as a new user, `POST /api/v1/accounts` "Amex", "Debit card", "Cash"
   → `color` `#e11d48`, `#0891b2`, `#a855f7`. Rename Amex → colour unchanged. `GET
   /api/v1/accounts` → every item has `color`.
2. **Freed colour**: delete the unused "Cash" → the next new account gets `#a855f7` again.
3. **Totals**: record October spendings on Amex (2) and Debit card (1);
   `GET /api/v1/account-totals?from=2026-10-01&to=2026-10-31` → two items, largest first, each
   with its account's `color`; `total` equals the sum of the items and equals
   `/api/v1/category-totals` for the same range and the October entry of `/api/v1/monthly-totals`.
4. **Only used accounts**: an account with no October spending is absent; an empty month → `items:
   []`, total `0.00`.
5. **Range rules**: `from` after `to`, or more than 366 days → `422` with the same pointers and
   messages as category totals; no token → `401`.
6. **Isolation**: a second user's spendings never appear in the first user's account totals.
7. **Back-fill**: covered by the migration IT (a user on V5 with 9 accounts → the 7 spare colours,
   then `#1e5b45`, `#c2410c`, in creation order).
8. **Performance**: `TotalsPerformanceIT` — 10,000 spendings, account totals for a month well
   under 1 s.

## 4. Frontend checks (`pnpm dev` against the local backend)

Desktop (wide window):

1. Below the By category card there is a **By account** card with its own Table / Pie toggle.
   Table: only October's accounts, swatch, amount, whole-number share, largest first, total row
   equal to the month total.
2. Switch By account to Pie: one slice per account in its colour, legend with name, amount and
   share; hover a slice → "Amex · $… · …%" at once and the slice is emphasised; Tab onto the
   slices → same label. By category's Table / Pie choice is unaffected; both survive a month
   change and a reload.
3. Edit a spending from Amex to Debit card → both By account figures change without a reload.
   Add an import → totals update.
4. Narrow the window until the summary cards drop below the table → By account moves with them.
5. An empty month → the short empty message in the card.

Phone (390 × 844, and 360 wide):

6. The switch reads "Spendings | By category | By account" on one row, no sideways scroll, each
   option at least 44px tall.
7. By account → Table / Pie toggle; with 8 accounts in a month the donut shows 6 slices plus
   "Everything else" equal to the other two, total in the middle; tap a slice → label; tap
   outside → hidden.
8. Choose Pie for By account and Table for By category → each keeps its choice; change month →
   both kept.
9. Empty month → "Nothing to break down in [Month Year]."

## 5. After deploy

- `homelab:hl-deploy-status` for both services (Argo CD Synced/Healthy).
- At https://budget.valyalkin.com: existing accounts each have a colour (none grey), By account
  matches the month total on desktop and phone.
- Add the 008 row to the README Features table.
