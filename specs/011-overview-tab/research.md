# Research: Overview tab

All decisions below were made against `origin/main` of both services (backend `782044a`, frontend
`c322164`) and the Overview artboard of the HL-Budget design canvas.

## R-001: No backend change; the year comes from existing endpoints

**Decision**: The Overview is built from five existing `/api/v1` operations, with no new
endpoint, field or migration:

| Needed for | Operation | Call |
|---|---|---|
| Spent per month, Spent tile, "today" | `listMonthlyTotals` | `GET /api/v1/monthly-totals?year=Y` |
| Earned per month, Earned tile | `listMonthlyIncomeTotals` | `GET /api/v1/monthly-income-totals?year=Y` |
| Spent by category | `listCategoryTotals` | `GET /api/v1/category-totals?from=Y-01-01&to=Y-12-31` |
| Spent by account | `listAccountTotals` | `GET /api/v1/account-totals?from=Y-01-01&to=Y-12-31` |
| Earliest year (FR-006) | `listSpendings`, `listIncomes` | `?sort=date,asc&size=1&page=0`, as `getEarliestYear` already does |

**Rationale**: The category and account totals take any range of at most 366 days, and a
calendar year is 365 or 366 days, so January 1 to December 31 is accepted. Both already return
only rows with spendings, largest first, ties by name (FR-017), with exact sums. The monthly
endpoints always return 12 months with zeros, and the backend refuses future-dated entries
(`EntryRules`: "must be today or earlier"), so totals for January 1 to December 31 of the
current year are the year to date. Year totals are the sum of the 12 monthly totals, which is
exact because they are decimal strings, summed in cents (`toCents`). Business rules stay where
they are; the frontend only adds and divides for display, as the month total card already does.

**Alternatives considered**:
- *A new `GET /api/v1/year-summary` endpoint* returning everything in one response: one call
  instead of four, but it duplicates four sums that already exist and adds a contract, tests and
  a release for no visible gain. At personal scale four parallel calls are well under SC-003.
  Rejected (Principle VI).
- *Computing totals from the raw entries in the frontend*: would page through up to thousands of
  entries and move sums into the frontend (Principle II). Rejected.

## R-002: A `/overview` route that keeps `?month=`

**Decision**: A new route `app/overview/page.tsx`. Its URL takes two optional parameters:

- `month=YYYY-MM`: the month the Spendings and Income tabs were showing. The tab links already
  carry it, so it is kept as is and handed back when the user returns (FR-002).
- `year=YYYY`: the year shown on Overview. When absent, the year of `month`; when that is absent
  too, the current year. Out of range (in the future or before the earliest year) is clamped, as
  a future `month` is clamped today.

The year arrows are links (`replace`, `scroll={false}`) that change only `year`, so Back never
steps through years, and Overview's year never changes `month` (FR-002). The Spendings and
Income links on Overview read `month` and drop `year`.

**Rationale**: It follows the 009 pattern: URL state, links that replace history, server-side
data load per request. A reload keeps the shown year.

**Alternatives considered**: client-only year state (lost on reload, needs client fetching
through a route handler, a second data path); writing the year back into `month` (would move the
other tabs, against FR-002).

## R-003: Tabs become Spendings | Income | Overview

**Decision**: Add a `Tab = Ledger | 'overview'` type and a `TABS` list (`spendings`, `income`,
`overview`) next to `LEDGERS` in `app/lib/ledger.ts`, with each tab's label and route.
`LedgerTabs` (desktop nav and phone segmented switch) and `AppHeader`'s `StaticTabs` iterate
`TABS` and take `current: Tab`. The segmented switch is already `grid auto-cols-fr`, so a third
option shares the width. At 320 px wide it gives each option about 96 px, still above the 44 px
target. `LEDGERS` stays two items, so nothing that loops over ledgers changes.

The Overview link is `/overview?month=…` when a month is selected. `LEDGER` is not given an
`overview` entry, because Overview is not a ledger.

## R-004: ECharts, tree-shaken, SVG renderer, in one client wrapper

**Decision**: Add `echarts@^6` to the frontend's dependencies (the user's choice), imported
through `echarts/core` with only `BarChart`, `GridComponent`, `TooltipComponent` and the
`SVGRenderer`. One client component, `app/ui/charts/echart.tsx`, owns the chart's whole life:
`init` on mount, `setOption(option, { notMerge: true })` when the option changes, a
`ResizeObserver` calling `resize()`, and `dispose()` on unmount. ECharts is imported with a
dynamic `import()` inside an effect, so it never runs on the server and stays out of the
Spendings and Income bundles. The option objects are built by pure functions in
`app/lib/overview-charts.ts`, which are unit tested without a browser.

**Rationale**: Only the Overview uses ECharts (spec Assumptions). The existing donuts are
unchanged. The core import with three components and one renderer keeps the added code to roughly
a third of the full build. The SVG renderer draws crisp text and bars at any pixel density, and
Playwright can read its output. Canvas is a little faster, but that only matters for far more
data points than 12 or about 20. ECharts uses no `eval` and only inline styles, which the
existing CSP already allows (`style-src 'self' 'unsafe-inline'`). The CSP needs no change.

**Alternatives considered**: `echarts-for-react` (another dependency for about 40 lines of
lifecycle code); the full `echarts` import (about 3× the bytes); canvas renderer (see above).

## R-005: Chart behaviour per the design

**Decision**: The look follows the design's `draw()` script:

- **Earned vs spent**: category x-axis Jan–Dec, value y-axis with `$Nk` labels, two bar series
  (Earned `#2a78d6`, Spent `#eb6834`), rounded tops, `barMaxWidth: 28`, axis-shadow tooltip with
  the month and both amounts to the cent. Future months of the current year are `null`, so
  ECharts draws no bar for them (FR-013).
- **Spent by category / by account**: horizontal bars, `yAxis` category `inverse: true` (largest
  on top), every label shown (`interval: 0`), all bars Spent orange, tooltip
  "Name<br>$7,680.00 · 24% of spent". Chart height is `max(220, rows × 30 + 40)` for categories
  and `rows × 48 + 40` for accounts, so every name stays readable (FR-017).
- **Tooltip style**: dark `#17191A` box, white 13 px text, the app's sans font.
- **Narrow widths**: below 480 px chart width the month labels become one letter (J, F, M…), the
  y-axis gets fewer ticks, and category labels longer than 14 characters are cut with "…". The
  tooltip always shows the full month and name.
- **Tap**: ECharts' default `triggerOn: 'mousemove|click'` shows the tooltip on tap. Tapping
  outside the chart hides it: a document `pointerdown` listener outside the chart dispatches
  `hideTip` (US5 scenario 4).
- **Animation**: 250 ms, and none when `prefers-reduced-motion: reduce`.

## R-006: Accessibility of the charts

**Decision**:
- The chart box is `role="img"` with an `aria-label` summary built by
  `app/lib/overview-charts.ts`. For Earned vs spent: "Column chart of money earned and spent in
  each month of 2026. Earned $61,000, spent $48,240." For a breakdown: "Bar chart of 2026
  spending by category. Largest is Rent at $25,800." ECharts' own `aria` option is off, so
  nothing is announced twice (FR-024).
- **Keyboard (FR-014, FR-018)**, changed during implementation: a transparent `role="slider"`
  (named "Months", "Categories" or "Accounts") lies over each chart with
  `pointer-events: none`, so the mouse and touch still reach the chart. On focus it highlights
  the first item. The arrow keys, Home and End move through the items with
  `dispatchAction({ type: 'showTip', seriesIndex: 0, dataIndex })`. The slider's
  `aria-valuetext` is that item's text, so screen readers read it out. Escape and blur hide the
  tooltip.

  The first plan was a focusable `role="img"` with a live region. jsx-a11y rejects focus and key
  handlers on an image, and a slider is the standard pattern for stepping through chart data.
- The Table view is the full accessible equivalent of each chart. Every view keeps the axe
  checks (SC-005).

## R-007: Remembered Chart / Table choice

**Decision**: Three `localStorage` keys, `hlb.overview.monthsView`,
`hlb.overview.categoryView` and `hlb.overview.accountView`, holding `'chart' | 'table'`, read
and written in try/catch with `'chart'` as the fallback, the same way the existing breakdown
cards store `hlb.byCategoryView` (FR-023). They are separate from the Spendings and Income keys.
To avoid a flash on load, the first render uses `'chart'` and switches after the stored value is
read, as the breakdown cards already do.

## R-008: One responsive tree; phone header and menu around it

**Decision**: The Overview content is a single responsive tree, matching the design: tiles in
`grid-cols-[repeat(auto-fit,minmax(220px,1fr))]`, the two breakdown cards in
`repeat(auto-fit,minmax(min(460px,100%),1fr))`. Below 640 px it collapses to one column on its
own. The shell around it differs:

- **Desktop** (`DesktopOnly`): `AppHeader` with `current="overview"` and the existing Settings
  button.
- **Phone** (`PhoneOnly`): `PhoneHeader` with the three-option switch and the account menu with
  Settings and Sign out. Import CSV is not offered on Overview, because there is nothing to
  import into. `AccountMenu` takes an `allowImport` prop, `true` by default, so Spendings and
  Income are unchanged.

`PhoneHeader` currently reads the ledger from `LedgerContext`. It gains an optional `current`
prop, so Overview can pass `'overview'` without a ledger context.

**Alternatives considered**: separate phone and desktop Overview trees (as for the ledgers). The
ledger pages differ a lot between phone and desktop: full-screen forms, a list instead of a
table. The Overview does not; it is cards that stack. Two trees would double the code for no
gain.

## R-009: Number formatting

**Decision**: Reuse `toCents`, `formatCents`, `sharePercent` from `app/lib/utils.ts`. Add
`formatWholeDollars(cents)`, which gives "$12,480" and "−$1,240" (minus sign U+2212, before the
dollar sign, FR-011), and `formatShare(part, total)`, which gives whole percentages and one
decimal below 1% (FR-022). The share uses the design's rule `n < 1 ? n.toFixed(1) :
Math.round(n)`. The design's `n < 0.1` check is a bug that the spec's wording corrects. Monthly
averages are `round(totalCents / months)`.

## R-010: Freshness and errors

**Decision**: `/overview` is a dynamic, per-request server page (it reads the session), like
`/spendings`. Client navigations to it fetch fresh data, because Next 16's dynamic router cache
time is 0. The existing server actions refresh the router after a change, so coming back from
another tab always shows current figures (FR-027). A failing load renders `app/overview/error.tsx`
with the shared `LoadError` and its Try again button, and shows no partial figures (FR-026).
`loading.tsx` shows a skeleton of the tiles and cards.
