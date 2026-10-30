# Quickstart: Import Spendings from CSV

How to prove feature 004 works, from gates to the deployed app. Details live in
[contracts/openapi.yaml](./contracts/openapi.yaml), [data-model.md](./data-model.md) and
[research.md](./research.md); this guide only runs them.

## 0. Prerequisites

- Both service repos on branch `004-csv-import`, cut from `origin/main` (R-015).
- Docker running (Testcontainers), Node 22 + pnpm, Java 21.
- Local `.env.local` files from features 001–003; no new variables.
- `KUBECONFIG=~/.kube/homelab.yaml` for the deploy checks.

## 1. Gates

```bash
# hl-budget-backend
./gradlew check            # ktlint, detekt, unit + ITs, Kover 80/75 overall, 75 per package

# hl-budget-frontend
pnpm generate:api          # after copying the backend's contracts/api.yaml
pnpm typecheck && pnpm lint && pnpm test
pnpm e2e                   # includes e2e/import-csv.spec.ts and the axe scans
```

Expected: all green. `contracts/api.yaml` in both repos contains `createSpendingImport` and
`getSpendingImport`, and no existing operation changed (`git diff` touches only additions).

## 2. Backend by hand

Run the backend locally (`./gradlew bootRun`) with a token for a test user (as in 003's
quickstart §2), then:

| Check | Request | Expect |
|---|---|---|
| Import two rows | `POST /api/v1/spending-imports` with the contract's `twoRows` example and `Idempotency-Key: k1` | `201`, `Location` set, `spendingCount: 2`, `lastDate: 2026-10-28`; both spendings in `GET /api/v1/spendings`; a new category/account only for names that didn't exist |
| Replay | the same request again | `200`, `Idempotent-Replayed: true`, same `id`; still 2 spendings |
| Key reused | same key, different rows | `422`, `details.reason: idempotency-key-reused` |
| All or nothing | 3 rows, row 1 dated tomorrow, row 2 amount `0` | `422` with `#/spendings/1/date` and `#/spendings/2/amount/amount`; no spending, category or account created |
| Too many | 1,001 rows | `422` on `#/spendings` |
| Read back | `GET` the `Location` | `200`, same summary; another user's token → `404` |

## 3. In the browser (signed in)

1. Spendings screen: "Import CSV" is above the month total card. Click + to open a row: the
   button becomes unavailable; cancel the row and it is available again.
2. Click "Import CSV". Check the window matches the design ("Import CSV window"), the page behind
   cannot be clicked, and Tab stays inside.
3. Click "Download sample CSV". `sample-spendings.csv` is saved and the window stays open. Open it
   and compare it with research R-011.
4. Drag the sample over the drop area (it highlights) and drop it. The review shows
   "sample-spendings.csv · 10 rows" and "3 rows need attention":
   - Dr. Patel Dental: category highlighted.
   - Birch Yoga Studio: account highlighted.
   - Thrift & Co.: amount highlighted.

   All sample dates are in October 2026 or earlier, so no row is flagged as a future date.

   Also check:
   - Ride share shows `18.90`.
   - Hardware Depot shows `1041.18` dated 3 October 2026.
   - "Lakeshore Fuel, Hwy 7" keeps its comma.
5. "Add spendings" is greyed out. Fill the three gaps using autofill: type `g` in a category and
   check it completes as on the Spendings screen. Tab moves along the row and on to the next row.
   The badge counts down, the footer total updates, and the button becomes "Add 10 spendings".
6. Remove one row: the row count, total and button label drop by one.
7. Press Escape in the review with no popup open: you are back at the pick window. Drop the file
   again: the review starts fresh, without your edits (US4 #3).
8. Fix the rows again and click "Add N spendings". The window closes, the screen shows the month
   of the latest date, and "Added N spendings from sample-spendings.csv" appears. The month line,
   table and cards include the spendings. A new category "Coffee" exists if it didn't before.
9. Refusals in the pick window, each showing its message from the spec with no review opening:
   - a `.png` file;
   - an empty `.csv`;
   - a header-only `.csv`;
   - a 2 MB `.csv`;
   - a 1,001-row `.csv`.
10. **SC-005**: import a generated 1,000-row file (`node e2e/fixtures/make-csv.mjs 1000 > big.csv`
    in the frontend repo; the helper is added by this feature). The review appears within 3 s, and typing in a field shows no visible
    lag.
11. **Failure**: stop the backend, click add. The review stays with all rows and a "Couldn't
    reach the server" message. Restart the backend and click add again: the spendings are added
    once.

## 4. Deploy

1. Merge and release the backend first. Flyway applies `V4__create_spending_imports.sql`. Run
   `homelab:hl-deploy-status` for `hl-budget-backend`: Synced/Healthy, `/readyz` 200.
2. Merge and release the frontend, then run `hl-deploy-status` for `hl-budget-frontend`.
3. Live check: repeat §3 steps 2, 4, 5 and 8 on the homelab URL with the sample file, then delete
   the imported spendings from the table.
4. Update the spec repo README Features table with feature 004 (small docs PR).
