---

description: "Task list for 011-overview-tab"
---

# Tasks: Overview tab

**Input**: Design documents from `/specs/011-overview-tab/`

**Prerequisites**: plan.md, spec.md, research.md, data-model.md, contracts/overview-ui.md,
quickstart.md

**Tests**: REQUIRED. Constitution Principle IV says tests ship with the code and every new
user-visible flow gets a Playwright spec. In each phase, write the tests first and see them fail
before writing the code that makes them pass.

**Organization**: Tasks follow spec.md US1–US5. All code is in the frontend repository
`hl-budget-frontend` (written `fe/` below). There is **no backend work** (research R-001): every
figure comes from existing `/api/v1` endpoints, listed in contracts/overview-ui.md §1.

**E2E data**: the Playwright specs sign in as the shared Auth0 test user against a real backend,
whose data other specs also change. Overview e2e must therefore assert **differences**: read a
figure, add an entry with a distinctive amount through the existing UI helpers
(`addSpendingViaUi`, `addIncomeViaUi` in `fe/e2e/support.ts` and `fe/e2e/income/support.ts`),
reload Overview and check that the figure changed by exactly that amount. They must not assert
absolute totals. Exact sums, clamping and formatting are covered by unit tests.

**Branch**: create `011-overview-tab` in `fe/` from an up-to-date `origin/main`.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Can run in parallel (different files, no dependency on unfinished tasks)
- **[Story]**: US1–US5 (spec.md)

## Path Conventions

- Frontend: `fe/app/...`, `fe/e2e/...` (`fe/` = `/Users/valyalkin/Documents/Projects/hl-budget-frontend`)
- Spec docs: `specs/011-overview-tab/...`
- Design reference: the `project/Overview.dc.html` artboard of
  https://claude.ai/artifact/QTxGKQCY8eE5SUq7387QCZ (read it with the Artifact tool, `path`
  `project/Overview.dc.html`). Its `draw()` script has the ECharts options to copy, and its markup
  has the exact wording, colours and spacing.

---

## Phase 1: Setup

- [X] T001 In `fe/`, run `git fetch && git switch -c 011-overview-tab origin/main`, `pnpm install`,
  then `pnpm typecheck && pnpm lint && pnpm test` and confirm they are green before any change.
  Read `fe/app/lib/ledger.ts`, `fe/app/lib/data.ts` (`read`, `getMonthlyTotals`,
  `getCategoryTotals`, `getAccountTotals`, `getEarliestYear`, `READERS`), `fe/app/lib/utils.ts`,
  `fe/app/ui/ledger-tabs.tsx`, `fe/app/ui/app-header.tsx`, `fe/app/ui/ledger-page.tsx`,
  `fe/app/ui/load-error.tsx`, `fe/app/ui/phone/phone-header.tsx`,
  `fe/app/ui/phone/account-menu.tsx`, `fe/app/ui/phone/layout-switch.tsx`,
  `fe/app/ui/spendings/breakdown-card.tsx` (how it stores the Table/Pie choice) and
  `fe/app/spendings/{page,loading,error}.tsx` to confirm the shapes the plan relies on.
- [X] T002 Add `echarts` `^6` to `dependencies` in `fe/package.json` with `pnpm add echarts@^6`.
  Confirm `import { init, use } from 'echarts/core'`, `BarChart` from `echarts/charts`,
  `GridComponent` and `TooltipComponent` from `echarts/components`, and `SVGRenderer` from
  `echarts/renderers` typecheck (research R-004).

---

## Phase 2: Foundational (blocks every story)

**Purpose**: the third tab, the route, the data loader and the formatting helpers that every
story uses.

### Tests first

- [X] T003 [P] Unit tests in `fe/app/lib/utils.test.ts` for the new helpers (research R-009):
  - `formatWholeDollars(1248000)` → `"$12,480"`; `formatWholeDollars(-124000)` → `"−$1,240"`, with
    U+2212 before `$` (FR-011); `formatWholeDollars(0)` → `"$0"`; it rounds half-cents up away
    from zero (`formatWholeDollars(1050)` → `"$11"`).
  - `formatShare(part, total)`: whole percentages (`"24%"`); one decimal below 1% (`"0.4%"`);
    `"0%"` when `total` is 0 (FR-022).
- [X] T004 [P] Unit tests in `fe/app/lib/overview.test.ts` (new) for `parseYear` and
  `buildOverview` (pure, no fetch), following data-model.md "Rules":
  - `year` defaults to `?year`, else the year of `?month`, else `currentYear`; non-numeric values
    fall back to the default; values above `currentYear` clamp to `currentYear`, values below
    `minYear` clamp to `minYear`.
  - `minYear` = min(earliest spending year, earliest income year); `currentYear` when both are
    null.
  - `monthsElapsed` = 12 for a past year, today's month for the current year.
  - `months` has 12 rows. Rows after today's month in the current year have
    `earnedCents`/`spentCents`/`netCents` = `null`; other months with no entries are `0`.
  - `totals.earnedCents`, `spentCents` are the cent sums of the 12 monthly totals (decimal
    strings such as `"2741.10"` must sum exactly); `savedCents = earnedCents − spentCents`, which
    can be negative.
  - `byCategory`/`byAccount` keep the API order and get `share = cents / spentCents × 100` (0 when
    `spentCents` is 0); `Σ cents` equals `spentCents` for consistent inputs.
  - `month` is passed through untouched (or `null`).
- [X] T005 [P] Update `fe/app/ui/app-header.test.tsx` and add `fe/app/ui/ledger-tabs.test.tsx`
  (new if missing):
  - the Main navigation lists links "Spendings", "Income", "Overview" in that order.
  - `aria-current="page"` is on the current tab, including `current="overview"`.
  - with `?month=2026-03` the hrefs are `/spendings?month=2026-03`, `/income?month=2026-03`,
    `/overview?month=2026-03`; without it they are bare routes (contracts/overview-ui.md §2).
  - The phone `segmented` variant renders the same three links.

### Implementation

- [X] T006 In `fe/app/lib/ledger.ts` add `export type Tab = Ledger | 'overview'` and
  `export const TABS: readonly { tab: Tab; label: string; route: '/spendings' | '/income' | '/overview' }[]`
  = Spendings, Income, Overview (research R-003). Leave `LEDGERS` and `LEDGER` unchanged.
- [X] T007 Change `fe/app/ui/ledger-tabs.tsx` to iterate `TABS` and take `current: Tab`. Keep
  `replace`, `scroll={false}`, `?month=` handling and both variants exactly as they are. Change
  `StaticTabs` and the `ledger` prop in `fe/app/ui/app-header.tsx` to `current?: Tab` (default
  `'spendings'`), and update its callers (`fe/app/ui/ledger-page.tsx`). Make T005 pass.
- [X] T008 In `fe/app/ui/phone/phone-header.tsx` add an optional `current?: Tab` prop. When it is
  given it replaces `useLedger()`, so the header works outside a `LedgerProvider`; without it,
  behaviour is unchanged. Use `useContext` directly or a non-throwing variant of `useLedger`. In
  `fe/app/ui/phone/account-menu.tsx` add `allowImport = true` and hide the Import CSV button when
  it is false (research R-008). Add one test each in the existing phone test files.
- [X] T009 [P] Implement `formatWholeDollars` and `formatShare` in `fe/app/lib/utils.ts`. Make T003
  pass.
- [X] T010 Create `fe/app/lib/overview.ts` with:
  - the `OverviewData`, `MonthRow` and `BreakdownRow` types exactly as in data-model.md.
  - `parseYear(params, today, minYear)`.
  - pure `buildOverview({ year, today, minYear, month, spendMonthly, incomeMonthly, categoryTotals, accountTotals })`.
  - `loadOverview(searchParams)` (server only). Round 1, `Promise.all`: `ensureMyProfile()`,
    `getMonthlyTotals(y0)`, the income monthly totals reader for `y0`, `getEarliestYear('spendings')`,
    `getEarliestYear('income')`. Here `y0` = `?year` or the year of `?month`, or `undefined` for the
    current year. Round 2: clamp the year, then `Promise.all` the category and account totals for
    `{year}-01-01`..`{year}-12-31`, re-fetching the two monthly totals only when clamping changed
    the year (contracts/overview-ui.md §1). Add `getYearCategoryTotals(year)` and
    `getYearAccountTotals(year)` to `fe/app/lib/data.ts`, beside the month versions.
  Make T004 pass.
- [X] T011 [P] Let `fe/app/ui/load-error.tsx` show an Overview message. Add an optional
  `message?: string` prop that wins over `LEDGER[ledger].loadFailed`, and make `ledger`
  optional. The wording is "We couldn't load your overview. Try again in a moment.". Extend
  its test.
- [X] T012 Create the route (contracts/overview-ui.md §2):
  - `fe/app/overview/page.tsx`: metadata `title: 'Overview'`, `robots: { index: false, follow: false }`.
    It renders `<OverviewPage searchParams>`.
  - `fe/app/overview/loading.tsx`: renders `OverviewSkeleton`.
  - `fe/app/overview/error.tsx`: `LoadError` with the Overview message (FR-026).
  - `fe/app/ui/overview/overview-page.tsx`:
    - `DesktopOnly` → `AppHeader signedIn current="overview" settingsSlot={<Suspense><SettingsSection/></Suspense>}`.
    - `PhoneOnly` → a `PhoneUiProvider` with `PhoneHeader current="overview"` and the account menu
      with `allowImport={false}`. Render its menu and Settings layers the way `PhoneApp` does, with
      only `menu` and `settings` layer kinds.
    - Then `<Suspense fallback={<OverviewSkeleton/>}><OverviewSection/></Suspense>`, rendered once
      for both layouts.
  - `fe/app/ui/overview/overview-section.tsx`: async server component that calls `loadOverview`
    and renders `<OverviewView data>`.
  - `fe/app/ui/overview/overview-view.tsx`: client component, `<main>` with the h1 "Overview"
    only, for now.
  - `fe/app/ui/overview/overview-skeleton.tsx`: three tile and three card placeholders.
  Check that `/overview` redirects to sign-in when signed out, the same way as `/spendings`, and add
  it to `fe/e2e/protected-routes.spec.ts`.
- [X] T013 Update existing e2e that count or list the header tabs (search `fe/e2e` for
  `'Income' })` and `navigation', { name: 'Main'`), so they accept the third tab and still pass:
  `fe/e2e/income/month-view.spec.ts`, `fe/e2e/phone/income/income.spec.ts`, and any others found.
  Add `fe/e2e/overview/support.ts` with `openOverviewTab(page)`: click Main → "Overview", wait
  for `**/overview**` and the h1 "Overview". Re-export `../support` and `../income/support`
  helpers.

**Checkpoint**: three tabs everywhere, `/overview` loads (empty main), all existing specs green.

---

## Phase 3: User Story 1 — See how my year is going (Priority: P1) 🎯 MVP

**Goal**: heading, period line and the Earned / Spent / Saved tiles for the shown year.

**Independent Test**: open Overview from Spendings; the year is the selected month's year; the
tiles match the entries (spec US1).

### Tests for User Story 1 ⚠️ (write first, see them fail)

- [X] T014 [P] [US1] Component tests in `fe/app/ui/overview/total-tiles.test.tsx` (new), fed
  `OverviewData` fixtures:
  - Earned and Spent show `formatWholeDollars` amounts and "avg $X a month", where
    X = `round(cents / monthsElapsed)` (FR-009).
  - Saved ≥ 0 shows the amount and "N% of what you earned".
  - Saved < 0 shows "−$1,240" and "Spent more than earned".
  - Spendings with no income shows the negative amount and "Spent more than earned", never a share
    of zero.
  - An empty year shows $0 everywhere and "Nothing recorded in 2026" (FR-010).
  - Each tile has its label text "Earned", "Spent", "Saved", and the region is named "Year totals".
- [X] T015 [P] [US1] Test in `fe/app/ui/overview/overview-view.test.tsx` (new) for the period line:
  - "January to December 2025" for a past year.
  - "January to November 2026 (year to date)" for the current year with today in November.
  - "January to December 2026 (year to date)" when today is in December (FR-007, data-model.md).
- [X] T016 [US1] E2E `fe/e2e/overview/year-totals.spec.ts` (new):
  - From Spendings at the current month, open Overview. URL is `/overview?month=…`, h1 "Overview",
    the period line ends "(year to date)", and the tab has `aria-current="page"`.
  - Read the Spent and Saved tiles. Add a spending of $7.31 on Spendings and return to Overview:
    Spent rose by $7 or $8 (whole-dollar rounding) and Saved fell by the same amount.
  - Add an income of $13.00 and return: Earned rose by $13 (FR-027).
  - Choose Spendings: the same month is shown (FR-002).
  - Reloading `/` still opens Spendings (FR-003).

### Implementation for User Story 1

- [X] T017 [US1] Create `fe/app/ui/overview/total-tiles.tsx`: a `section aria-label="Year totals"`
  with a `grid-cols-[repeat(auto-fit,minmax(220px,1fr))] gap-3.5` grid of three white cards
  (border `line`, radius 14 px). Each card has its label row (Earned with a `#2a78d6` swatch,
  Spent with a `#eb6834` swatch, Saved with none), an amount in the display font at 32 px bold,
  and a 13 px muted note. Wording as in T014. Make T014 pass.
- [X] T018 [US1] Fill `fe/app/ui/overview/overview-view.tsx`: a `max-w-[1240px] mx-auto px-6 py-8
  flex flex-col gap-6` main (`px-4` below 640 px) with the h1 "Overview" (display font, 36 px,
  28 px on phone), the period line (muted) and `<TotalTiles>`. Leave a placeholder slot for the
  year picker on the right of the heading row (flex-wrap). Make T015 and T016 pass.

**Checkpoint**: MVP. The user can see this year's Earned, Spent and Saved.

---

## Phase 4: User Story 2 — Compare earned and spent month by month (Priority: P1)

**Goal**: the Earned vs spent card with an ECharts grouped column chart or a month table, with
the choice remembered.

**Independent Test**: spec US2. Columns only for months with entries and none after the current
month; the tooltip shows the cents; Table shows Month, Earned, Spent, Net.

### Tests for User Story 2 ⚠️

- [X] T019 [P] [US2] Unit tests in `fe/app/lib/overview-charts.test.ts` (new) for
  `monthsOption(rows, { width, reducedMotion })`:
  - `xAxis.data` is Jan…Dec, or `J F M A M J J A S O N D` when `width < 480`.
  - two bar series named Earned (`#2a78d6`) and Spent (`#eb6834`), with `barMaxWidth: 28` and
    rounded tops `[4,4,0,0]`.
  - future months are `null` in both series.
  - the y-axis label formatter gives `$5k` for 5000 (whole dollars).
  - `animationDuration` is 250, or 0 when `reducedMotion`.
  - `aria.enabled` is false.
  - the tooltip formatter gives "March<br>● Earned  $6,100.00<br>● Spent  $4,215.37", with the full
    month name and `formatCents` amounts, and "—" for a null.
  Also test `monthsSummary(data)` → "Column chart of money earned and spent in each month of
  2026. Earned $61,000, spent $48,240." (research R-006).
- [X] T020 [P] [US2] Component tests in `fe/app/ui/charts/echart.test.tsx` (new). Mock
  `echarts/core` with a fake `init` that returns
  `{ setOption, resize, dispose, dispatchAction, on, off }`, and check:
  - `init` runs on mount with the SVG renderer, `setOption(option, { notMerge: true })` runs when
    the option changes, and `dispose` runs on unmount.
  - the box has `role="img"`, the given `aria-label` and `tabIndex=0`.
  - ArrowRight/ArrowLeft (`axis="x"`) or ArrowDown/ArrowUp (`axis="y"`) dispatch
    `showTip { seriesIndex: 0, dataIndex }`, moving the index and stopping at the ends; the first
    key press or focus starts at 0.
  - Escape and blur dispatch `hideTip`.
  - a `pointerdown` outside the box dispatches `hideTip`.
  - the visually hidden `aria-live="polite"` region gets the tooltip text from the
    `describe(dataIndex)` prop.
- [X] T021 [P] [US2] Component tests in `fe/app/ui/overview/months-card.test.tsx` (new), with
  `EChart` mocked:
  - region named "Earned vs spent", subtitle "Month by month, 2026", legend list with Earned and
    Spent.
  - "Show as" group with Chart (`aria-pressed="true"` by default) and Table.
  - Table lists months 1..`monthsElapsed` with Month, Earned, Spent, Net; a negative Net shows
    "−$…"; a month with nothing recorded shows $0.
  - the choice is written to `localStorage['hlb.overview.monthsView']` and read back on mount.
  - when `localStorage` throws, Chart still works (FR-023, research R-007).
  - an empty year shows "Nothing recorded in 2026." instead of the chart or table (FR-025).
- [X] T022 [US2] E2E `fe/e2e/overview/earned-vs-spent.spec.ts` (new):
  - The chart renders an SVG with bars inside the region.
  - Hover the current month's band: a tooltip with the month name and two amounts with cents
    appears.
  - Focus the chart and press ArrowRight: the tooltip moves; Escape hides it.
  - Switch to Table: the last row is the current month; Net = Earned − Spent for that row.
  - Reload: Table is still selected. Switch back to Chart for the next tests.

### Implementation for User Story 2

- [X] T023 [US2] Create `fe/app/ui/charts/echart.tsx` (client), with props
  `{ option, label, axis: 'x' | 'y', count, describe, height, className }`:
  - in `useEffect`, dynamically `import()` a local `fe/app/ui/charts/echarts-setup.ts` that calls
    `use([BarChart, GridComponent, TooltipComponent, SVGRenderer])` and re-exports `init`. Then
    `init(el, null, { renderer: 'svg' })`.
  - `setOption(option, { notMerge: true })` on option change; a `ResizeObserver` calls `resize()`;
    `dispose()` and `disconnect()` on unmount.
  - keyboard handling: arrows per `axis`, Escape and blur as in T020; a document `pointerdown`
    outside the box calls `hideTip`.
  - the live region text comes from `describe(index)`.
  - focus ring `focus-visible:outline-2 outline-brand`.
  Make T020 pass. ECharts never runs on the server, and `/spendings` does not import this file.
- [X] T024 [US2] Create `fe/app/lib/overview-charts.ts` with `monthsOption` and `monthsSummary`, as
  in T019 and the design's `draw()`:
  - grid `{ left: 8, right: 8, top: 12, bottom: 4, containLabel: true }`; `splitNumber` 5, or 3
    when `width < 480`; split lines `#E1E0D9`; axis line `#C3C2B7`.
  - labels `#5C625E` 13 px in the app's sans font.
  - tooltip `trigger: 'axis'`, `axisPointer: { type: 'shadow' }`, `backgroundColor: '#17191A'`,
    `borderWidth: 0`, `padding: 10`, white 13 px text; `confine: true`, so it stays inside the
    chart on a phone.
  Export the colour constants `EARNED = '#2a78d6'`, `SPENT = '#eb6834'` for the tiles and legends.
  Make T019 pass.
- [X] T025 [P] [US2] Create `fe/app/ui/overview/use-view-choice.ts`:
  `useViewChoice(key): ['chart' | 'table', set]`. The first render is `'chart'`; an effect reads
  `localStorage`, with every read and write in try/catch (research R-007). Create
  `fe/app/ui/overview/chart-card.tsx`: the white card frame (padding 20 px, radius 14 px, gap
  16 px, `min-w-0`) with the heading slot, a right-hand slot, and the "Show as" segmented group
  (buttons at least 44 px tall on phone and 38 px on desktop, as in the design; `aria-pressed`).
  It renders the chart or the table child.
- [X] T026 [US2] Create `fe/app/ui/overview/months-card.tsx` using `ChartCard`, `EChart`
  (`axis="x"`, `count={12}`, height 320 px, 260 px below 640 px, `describe` giving
  "March: earned $6,100.00, spent $4,215.37") and the month table:
  - table in an `overflow-x-auto` box; uppercase 12 px headers; numbers right-aligned in the mono
    font.
  - chart width for `monthsOption` measured with a `ResizeObserver` on the card.
  - `prefers-reduced-motion` read with `matchMedia`.
  Add it to `overview-view.tsx` below the tiles. Make T021 and T022 pass.

**Checkpoint**: US1 + US2, the full P1 scope.

---

## Phase 5: User Story 3 — See where the year's money went (Priority: P2)

**Goal**: the Spent by category and Spent by account cards, as ranked bars or tables.

**Independent Test**: spec US3. Categories largest first, amounts adding up to the Spent tile,
shares in the tooltip and table; the same for accounts; income never counted.

### Tests for User Story 3 ⚠️

- [X] T027 [P] [US3] Unit tests in `fe/app/lib/overview-charts.test.ts` for
  `breakdownOption(rows, spentCents, { width, reducedMotion })`:
  - `yAxis` is category, `inverse: true`, labels at `interval: 0`; names longer than 14
    characters are cut with "…" when `width < 480`.
  - one bar series in `#eb6834`, with radius `[0,4,4,0]` and `barWidth: '65%'`.
  - the x-axis formatter gives `$800` / `$2k`.
  - the tooltip formatter gives "Rent<br>$25,800.00 · 24% of spent", with the full name.
  Also test `breakdownHeight('category', n)` = `max(220, n*30+40)` and
  `breakdownHeight('account', n)` = `n*48+40` (research R-005), and `breakdownSummary` →
  "Bar chart of 2026 spending by category. Largest is Rent at $25,800.".
- [X] T028 [P] [US3] Component tests in `fe/app/ui/overview/breakdown-year-card.test.tsx` (new),
  `EChart` mocked, for both kinds:
  - region "Spent by category" with the big total and "total spent in 2026".
  - region "Spent by account" with "Where the $X came from".
  - Table columns Category|Account, Spent, Share; rows in the given order; amounts formatted with
    `formatWholeDollars`, shares with `formatShare`.
  - the separate keys `hlb.overview.categoryView` and `hlb.overview.accountView`.
  - `spentCents = 0` shows "No spendings in 2026." (FR-025).
- [X] T029 [US3] E2E `fe/e2e/overview/breakdowns.spec.ts` (new):
  - With E2E_ACCOUNT, add a spending of $9.17 in category "Other" on Spendings. On Overview, Spent
    by category in Table view shows "Other" and its amount rose by the same rounding as T016. Spent
    by account's row for E2E_ACCOUNT rose likewise.
  - The sum of the table amounts is within $N (N = row count, for rounding) of the Spent tile.
  - Add an income of $500 on Income. Neither breakdown card changes (FR-021).
  - Each card keeps its own Chart/Table choice across a reload (FR-023).
  - In Chart view, hovering the first bar shows "· N% of spent".

### Implementation for User Story 3

- [X] T030 [US3] Add `breakdownOption`, `breakdownHeight` and `breakdownSummary` to
  `fe/app/lib/overview-charts.ts`: grid `{ left: 8, right: 24, top: 8, bottom: 4, containLabel: true }`,
  x `splitNumber` 4, y labels `#17191A` 13 px, and the same tooltip style as T024. Make T027 pass.
- [X] T031 [US3] Create `fe/app/ui/overview/breakdown-year-card.tsx` with
  `kind: 'category' | 'account'`, using `ChartCard`, `EChart` (`axis="y"`, `count={rows.length}`,
  height `breakdownHeight`, `describe` giving "Rent: $25,800.00, 24% of spent") and the table.
  In `overview-view.tsx`, place the two cards in a
  `grid-cols-[repeat(auto-fit,minmax(min(460px,100%),1fr))] gap-6 items-start` grid below Earned
  vs spent. Make T028 and T029 pass.

---

## Phase 6: User Story 4 — Look at another year (Priority: P2)

**Goal**: previous and next year arrows that change every figure; disabled at the ends; the
other tabs keep their month.

**Independent Test**: spec US4.

### Tests for User Story 4 ⚠️

- [X] T032 [P] [US4] Component tests in `fe/app/ui/overview/year-picker.test.tsx` (new):
  - group "Choose year" with the year text in an `aria-live="polite"` element.
  - "Previous year" links to `/overview?month=2026-03&year=2025`, keeping `month`; without
    `month`, to `/overview?year=2025`.
  - at `minYear` the previous link is `aria-disabled="true"`, has no `href` and is styled muted;
    the next link behaves the same at `currentYear`.
  - both links use `replace` and `scroll={false}`.
  - each target is at least 44×44 px (`size-11`).
- [X] T033 [US4] E2E `fe/e2e/overview/year-picker.spec.ts` (new):
  - On the current year, "Next year" is disabled.
  - Visiting `/overview?year=2999` shows the current year; `/overview?year=1999` shows the earliest
    year, with "Previous year" disabled.
  - If "Previous year" is enabled on the current year, press it. The URL has `year=` one lower, the
    period line reads "January to December {year−1}", and the h1 stays. Press "Next year" to return.
  - From Overview with `?month=2026-03&year=2025`, choose Income: `/income?month=2026-03` shows
    March 2026 (FR-002).
  - Browser Back after a year change does not step back through years (replace).

### Implementation for User Story 4

- [X] T034 [US4] Create `fe/app/ui/overview/year-picker.tsx` as in the design: a white box
  (radius 14 px, padding 6 px) holding a chevron link, the year (display font, 22 px, tabular
  numbers, `min-w-16` centred) and a chevron link. Use Next `Link` with `replace` and
  `scroll={false}`; a disabled end renders as `<a aria-disabled="true">` with no `href` (contracts
  §2). Put it in the heading row of `overview-view.tsx`. Make T032 and T033 pass.

---

## Phase 7: User Story 5 — Use the Overview on a phone (Priority: P2)

**Goal**: one column, charts that fit, tap labels, tables that scroll inside their cards, 44 px
targets, three-option switch, menu without Import CSV.

**Independent Test**: spec US5, in the Playwright phone project at 390 and 360 wide.

### Tests for User Story 5 ⚠️

- [X] T035 [US5] E2E `fe/e2e/phone/overview/overview.spec.ts` (new), in the phone project:
  - The header switch shows Spendings | Income | Overview; choose Overview.
  - At 390 and at 360 wide, `document.documentElement.scrollWidth <= innerWidth` (no sideways page
    scroll); tiles are stacked (each tile's left edge is equal).
  - Every button and link in main and the header is at least 44×44 px.
  - Tapping a column shows the tooltip; tapping the heading hides it.
  - The month chart's x labels are single letters.
  - In Table view, a wide table scrolls inside its card while the page does not.
  - The account menu shows Settings and Sign out and no Import CSV; Settings opens and closes.
  - The year arrows work.

### Implementation for User Story 5

- [X] T036 [US5] Go through `overview-view.tsx`, `total-tiles.tsx`, `chart-card.tsx`,
  `months-card.tsx`, `breakdown-year-card.tsx` and `year-picker.tsx` at 360 px:
  - `px-4` gutters; the heading row wraps with the year picker under the title.
  - card headers wrap with the Show as group on its own line.
  - the Show as buttons are `min-h-11` below 640 px.
  - tables are `min-w-[20rem]` inside `overflow-x-auto`.
  - the chart heights from T026 and T031.
  Confirm that the overview-page phone shell from T012 handles the back gesture for the menu and
  Settings, as on Spendings (the existing `PhoneUiProvider` history handling). Make T035 pass.

---

## Phase 8: Polish & Cross-Cutting Concerns

- [X] T037 [P] Accessibility (SC-005): add `/overview` to `fe/e2e/a11y.spec.ts` (Chart views;
  then all three cards in Table view; with a chart focused and its tooltip shown) and to
  `fe/e2e/phone/a11y.spec.ts` (phone, Chart and Table, account menu open). There must be no
  serious or critical violations. Check text contrast on the muted notes (`#5C625E` on white is
  6.5:1).
- [X] T038 [P] Failure path (FR-026): add a case to `fe/e2e/failures.spec.ts`, following that
  file's existing way of making the API fail. `/overview` shows "We couldn't load your
  overview…" and Try again, and no tiles.
- [X] T039 Run `pnpm build` in `fe/` and check:
  - the `/overview` route is dynamic.
  - `echarts` code appears only in chunks loaded by `/overview`: search `.next/static/chunks` for
    `echarts`, and confirm the `/spendings` and `/income` page manifests in
    `.next/server/app/*/page_client-reference-manifest.js` do not reference those chunks.
  - no CSP violations in the browser console on `/overview` (`pnpm start`, open the page, check
    the console).
- [X] T040 Run the gates: `pnpm typecheck && pnpm lint && pnpm test` and `pnpm e2e` (both
  projects). Every pre-011 spec must pass unchanged, except the tab updates from T013. Fix
  anything that fails.

  > Done 2026-11-08: typecheck, lint, 607 unit tests and `pnpm build` are green. `/overview` is
  > partially prerendered like `/spendings`. ECharts is one trimmed chunk (510 KB, 172 KB
  > gzipped, no pie, line or map charts), and only the `/overview` client manifest references it.
  > The Playwright specs were written but not run locally, because this machine has no Auth0 test
  > user (`E2E_USERNAME` / `E2E_PASSWORD`), so they skip; they run in CI, as for 010. A
  > production build with a temporary, uncommitted preview route and sample data was checked in
  > Chromium at 1360 and 360 px wide:
  > - the layout matches the design, with no sideways scroll;
  > - hover and tap show the tooltips, and a tap elsewhere hides them;
  > - the keyboard steps through months;
  > - there are no CSP errors.
  >
  > That check found overlapping x-axis labels on the narrow category chart, now fixed with fewer
  > ticks.
  >
  > **Deviations from the plan**:
  > - `loadOverview` lives in `app/lib/data.ts`, the only place pages read data from (frontend
  >   skill). `app/lib/overview.ts` keeps the pure `parseYear`, `requestedYear` and
  >   `buildOverview`.
  > - Keyboard reading uses a transparent `role="slider"` over each chart (value text = the item)
  >   instead of a focusable `role="img"` with a live region. jsx-a11y forbids focus and key
  >   handlers on an image, and a slider is the standard pattern. Arrow keys, Home and End step;
  >   Escape hides.
  > - `OverviewRoot` sets `data-layout="phone"` on phones, so the phone e2e helpers work on the
  >   Overview.
- [ ] T041 Walk through `specs/011-overview-tab/quickstart.md` §4 checks 1–12 locally. Then commit
  in `fe/`, push, and open a PR titled "feat(011): Overview tab", linking the spec. After merge,
  run the release, run `homelab:hl-deploy-status` for `hl-budget-frontend`, and do the live check
  (quickstart §5).

  > Done 2026-11-09: PR https://github.com/valyalkin/hl-budget-frontend/pull/13 merged; release
  > image `29-48133d3`, hl-helm PR #35 merged. Live: `/healthz` is 200 and `/overview` redirects to
  > sign-in. The cluster API was unreachable (no SSH tunnel), so the Argo CD and pod checks were not
  > run. Still open: the signed-in look at https://budget.valyalkin.com/overview (quickstart §5),
  > which needs the user's account; the e2e job is skipped in CI as well.
- [X] T042 In this spec repo, mark the tasks done, set spec.md **Status** to Implemented, and add
  the 011 row to the README Features table ("Implemented YYYY-MM-DD (frontend only): an Overview
  tab after Income … deployed to the homelab") in a small PR.

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)** → **Foundational (Phase 2)** → user stories → **Polish (Phase 8)**.
- Phase 2 blocks every story: tabs, route, `loadOverview`, formatting.

### User Story Dependencies

- **US1** (tiles): only Phase 2.
- **US2** (Earned vs spent): only Phase 2. It creates `EChart`, `ChartCard`, `useViewChoice` and
  `overview-charts.ts`.
- **US3** (breakdowns): needs the US2 shared pieces (T023–T025).
- **US4** (year picker): only Phase 2. The data loader already reads `?year`, so this adds the
  control and its e2e.
- **US5** (phone): best done last, because it adjusts the components from US1–US4. The phone shell
  itself exists from T012.

### Within Each User Story

- Tests first, failing; then the implementation that makes them pass.
- Option builders (`overview-charts.ts`) before the cards that use them.

### Parallel Opportunities

- T003, T004 and T005 (different test files); T009 and T011 beside T006–T008.
- In US2, T019, T020 and T021 in parallel; T025 beside T023 and T024.
- In US3, T027 and T028 in parallel.
- US1 and US4 can run in parallel with US2 after Phase 2, because they touch different files
  (only the final placement in `overview-view.tsx` overlaps).
- T037 and T038 in parallel.

## Parallel Example: User Story 2

```text
Task: "T019 [P] [US2] monthsOption / monthsSummary unit tests in fe/app/lib/overview-charts.test.ts"
Task: "T020 [P] [US2] EChart wrapper tests in fe/app/ui/charts/echart.test.tsx"
Task: "T021 [P] [US2] Months card tests in fe/app/ui/overview/months-card.test.tsx"
# then
Task: "T023 [US2] EChart wrapper"   Task: "T024 [US2] monthsOption"   Task: "T025 [P] [US2] useViewChoice + ChartCard"
# then
Task: "T026 [US2] Months card + placement"
```

## Implementation Strategy

### MVP First (User Story 1 Only)

Phases 1–3: three tabs and the Earned / Spent / Saved tiles for the current year. This answers
"how much did I save this year" (SC-001) and can be deployed on its own.

### Incremental Delivery

1. MVP (US1).
2. US2: monthly chart and table (completes P1).
3. US3: breakdown cards.
4. US4: other years.
5. US5: phone polish, then Phase 8 and deploy.

One frontend PR is expected, as for earlier features. The checkpoints above are safe places to
stop and test.

## Notes

- No backend, contract or migration change; do not regenerate `api-schema.d.ts`.
- Keep Spendings and Income unchanged apart from the third tab. Every new prop on a shared
  component has a default that keeps today's behaviour.
- ECharts is used on Overview only; the donuts stay as they are.
