# Quickstart: validating feature 011

How to prove the feature works. The URL and screen structure are in
[contracts/overview-ui.md](./contracts/overview-ui.md), the derived figures and rules in
[data-model.md](./data-model.md), and the decisions in [research.md](./research.md).

## 1. Prerequisites

- Node 22 + pnpm. The backend is unchanged: run it from `origin/main` (`./gradlew bootRun`, as in
  003's quickstart) or point the frontend at the homelab API as before.
- Branch `011-overview-tab` in `hl-budget-frontend`, from `origin/main` (includes feature 010).
  No backend branch.

## 2. Gates

```sh
# hl-budget-frontend
pnpm install               # adds echarts
pnpm typecheck && pnpm lint && pnpm test
pnpm e2e                   # desktop and phone projects with axe; every pre-011 spec green
pnpm build                 # check the /overview route builds and echarts is not in the /spendings chunks
```

## 3. Test data

As one test user, record through the app or the API:

- **Previous year** (for example 2025): income of $5,800 in each of the 12 months; spendings in
  at least four categories on two accounts, in several months.
- **Current year**: income in January, March and October; spendings in January, March, October
  and one month with spendings only; nothing in the other months.

## 4. Checks

1. **Tab and route**: on Spendings at `?month=2026-03`, choose Overview. The header shows
   Spendings | Income | Overview with Overview selected; the URL is `/overview?month=2026-03`;
   the year is 2026. Choose Spendings: March 2026 is shown (FR-001, FR-002).
2. **Tiles**: Earned, Spent and Saved match the sums of the recorded entries; the averages divide
   by today's month number; the period line reads "January to {current month} 2026 (year to
   date)" (US1).
3. **Saved negative**: in a year where spendings are larger than income, Saved shows "−$…" and
   "Spent more than earned" (FR-010, FR-011).
4. **Earned vs spent**: chart has bars only in the months with entries and none after the current
   month; hover a month to see both amounts to the cent; switch to Table and check one row per
   month to date with Net (US2).
5. **Breakdowns**: Spent by category lists the categories largest first; its amounts add up to
   the Spent tile; hover shows "$… · N% of spent"; same for Spent by account; income never
   appears (US3).
6. **Year picker**: press Previous year. The URL gets `&year=2025` and every figure changes to
   2025, with the period line "January to December 2025". Next year returns to 2026, where Next
   is disabled. At the earliest year, Previous is disabled. A user with no entries sees only the
   current year (US4).
7. **Remembered views**: set Table on Spent by account, change year, go to Income and back,
   reload. It stays Table while the other cards stay Chart (FR-023).
8. **Freshness**: add a spending on Spendings, go to Overview, and check that Spent includes it
   without a reload (FR-027).
9. **Keyboard**: Tab to the Earned vs spent chart, then use Right arrow. The tooltip moves month
   by month, and Escape hides it. Do the same on a breakdown chart with Down arrow (FR-014,
   FR-018).
10. **Phone** (Playwright phone project at 390 and 360 wide, or a real phone at
    https://budget.valyalkin.com): the switch shows Spendings | Income | Overview; there is no
    sideways page scroll; tiles and cards are stacked; tapping a bar shows its tooltip and tapping
    elsewhere hides it; tables scroll inside their card; the account menu has Settings and Sign
    out but not Import CSV (US5).
11. **Errors**: stop the backend and open `/overview`. You see the error message and Try again,
    with no figures (FR-026).
12. **Accessibility**: axe has no serious or critical issues on Overview, desktop and phone, in
    Chart and Table views (SC-005).

## 5. Live check after deploy

Run `homelab:hl-deploy-status` for `hl-budget-frontend`, then repeat checks 1, 2, 6 and 10 at
https://budget.valyalkin.com with real data. Then update the README Features table.
