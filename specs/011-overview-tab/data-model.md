# Data model: Overview tab

Nothing new is stored: no migration, no table, no column, no API field (research R-001). This
file describes the frontend's view model for Overview, which is derived from four existing
responses on each load.

## Inputs (existing API shapes, unchanged)

| Response | Fields used |
|---|---|
| `MonthlyTotals` (spendings), `GET /monthly-totals?year=Y` | `year`, `today`, `months[].month`, `months[].total.amount`, `months[].spendingCount` |
| `MonthlyTotals` (income), `GET /monthly-income-totals?year=Y` | the same; `spendingCount` counts income entries |
| `CategoryTotals`, `GET /category-totals?from=Y-01-01&to=Y-12-31` | `total.amount`, `items[].category.{id,name}`, `items[].total.amount` |
| `AccountTotals`, `GET /account-totals?from=Y-01-01&to=Y-12-31` | `total.amount`, `items[].account.{id,name}`, `items[].total.amount` |
| `SpendingPage` / `IncomePage`, first item ascending | `items[0].date` → earliest year |

All money is converted to integer cents with `toCents` before anything is summed or compared.

## `OverviewData` (server → client, built in `app/lib/overview.ts`)

```text
OverviewData
├── year: number                 the shown year, clamped to [minYear, currentYear]
├── currentYear: number          from `today`
├── monthsElapsed: number        12 for a past year; today's month for the current year (1..12)
├── minYear: number              min(earliest spending year, earliest income year), or currentYear when none
├── month: string | null         the `?month=` passed through for the tab links (R-002)
├── currency: 'CAD'
├── totals
│   ├── earnedCents: number      Σ income months
│   ├── spentCents: number       Σ spending months
│   └── savedCents: number       earnedCents − spentCents (may be negative)
├── months: MonthRow[12]
├── byCategory: BreakdownRow[]   largest first, as the API returns it
└── byAccount: BreakdownRow[]    largest first, as the API returns it

MonthRow
├── month: 1..12
├── earnedCents: number | null   null when the month is after today's month in the current year
├── spentCents:  number | null   (same rule)
└── netCents:    number | null   earned − spent; null with the two above

BreakdownRow
├── id: string
├── name: string
├── cents: number
└── share: number                cents / spentCents × 100 (0 when spentCents is 0)
```

## Rules (from the spec)

| Rule | Source |
|---|---|
| `year` defaults to `?year`, else the year of `?month`, else `currentYear`; a value outside `[minYear, currentYear]` or not a 4-digit number is clamped or replaced by `currentYear` | FR-002, FR-005, FR-006, R-002 |
| Previous arrow enabled iff `year > minYear`; next arrow enabled iff `year < currentYear` | FR-006 |
| Period line: `January to December {year}` when `monthsElapsed = 12` and `year < currentYear`; otherwise `January to {Month} {year} (year to date)` (December of the current year also reads "year to date") | FR-007 |
| Averages: `round(earnedCents / monthsElapsed)`, `round(spentCents / monthsElapsed)` | FR-009 |
| Saved note: `earnedCents = 0 && spentCents = 0` → "Nothing recorded in {year}"; `savedCents < 0` → "Spent more than earned"; otherwise `formatShare(savedCents, earnedCents)` + " of what you earned" | FR-010 |
| Months table rows: months `1..monthsElapsed`; a month with no entries shows $0 | FR-015 |
| `Σ byCategory.cents = Σ byAccount.cents = spentCents` (checked in the data unit test; true by construction of the API) | FR-019, SC-002 |
| Empty states: `earnedCents = 0 && spentCents = 0` → Earned vs spent card shows "Nothing recorded in {year}."; `spentCents = 0` → both breakdown cards show "No spendings in {year}." | FR-025 |

## View state (client only)

| State | Values | Kept in |
|---|---|---|
| Earned vs spent view | `'chart'` (default) / `'table'` | `localStorage['hlb.overview.monthsView']` |
| Spent by category view | `'chart'` / `'table'` | `localStorage['hlb.overview.categoryView']` |
| Spent by account view | `'chart'` / `'table'` | `localStorage['hlb.overview.accountView']` |
| Tooltip / keyboard highlight | data index or none | component state only |
