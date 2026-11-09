# Contract: Overview tab

There is **no API contract change** in this feature: no new operation, parameter or field, so no
OpenAPI delta and no type regeneration (research R-001). This file fixes the two interfaces the
feature does add: the `/overview` URL and the screen's labelled structure that tests rely on.

## 1. API operations consumed (unchanged, `/api/v1`)

| operationId | Request on Overview |
|---|---|
| `ensureMyProfile` | `PUT /users/me` (as every page) |
| `listMonthlyTotals` | `GET /monthly-totals?year={Y}` |
| `listMonthlyIncomeTotals` | `GET /monthly-income-totals?year={Y}` |
| `listCategoryTotals` | `GET /category-totals?from={Y}-01-01&to={Y}-12-31` |
| `listAccountTotals` | `GET /account-totals?from={Y}-01-01&to={Y}-12-31` |
| `listSpendings` | `GET /spendings?sort=date,asc&size=1&page=0` (earliest year) |
| `listIncomes` | `GET /incomes?sort=date,asc&size=1&page=0` (earliest year) |

Load order: round 1 runs the profile, both monthly totals for the requested year, and both
earliest-year reads in parallel. Round 2 runs both breakdown totals for the clamped year. When
clamping changed the year, round 2 also reruns the two monthly totals. That makes at most two
round trips per load.

## 2. Route

`GET /overview[?month=YYYY-MM][&year=YYYY]`, signed in only (same guard as `/spendings`),
`robots: noindex, nofollow`, page title "Overview".

| Parameter | Meaning |
|---|---|
| `month` | Passed through untouched; the Spendings and Income tab links use it (`/spendings?month=…`) |
| `year` | The year shown; default the year of `month`, else the current year; clamped to `[minYear, currentYear]` |

Links on the page:

| Control | Target | History |
|---|---|---|
| Spendings / Income tab | `/spendings?month={month}` / `/income?month={month}` (no `month` when absent) | replace |
| Overview tab on Spendings / Income | `/overview?month={month}` | replace |
| Previous year | `/overview?month={month}&year={year−1}`, rendered `aria-disabled="true"` with no `href` at `minYear` | replace, no scroll |
| Next year | `/overview?month={month}&year={year+1}`, likewise at `currentYear` | replace, no scroll |

## 3. Screen structure (accessible names used by tests)

```text
banner
└── navigation "Main": links "Spendings", "Income", "Overview" (aria-current="page")
main
├── heading level 1 "Overview"; paragraph: period line (FR-007)
├── group "Choose year": link "Previous year", text {year} (aria-live polite), link "Next year"
├── region "Year totals"
│   ├── "Earned" tile: amount, "avg $X a month"
│   ├── "Spent" tile: amount, "avg $X a month"
│   └── "Saved" tile: amount, note (FR-010)
├── region "Earned vs spent" (heading level 2), subtitle "Month by month, {year}"
│   ├── list "Legend": Earned, Spent
│   ├── group "Show as": buttons "Chart" / "Table" (aria-pressed)
│   └── chart: img with summary label, focusable | table: Month, Earned, Spent, Net | empty message
├── region "Spent by category" (heading level 2), total + "total spent in {year}"
│   ├── group "Show as": "Chart" / "Table"
│   └── chart (img) | table: Category, Spent, Share | "No spendings in {year}."
└── region "Spent by account" (heading level 2), "Where the {total} came from"
    ├── group "Show as": "Chart" / "Table"
    └── chart (img) | table: Account, Spent, Share | "No spendings in {year}."
```

Wording, colours (Earned `#2a78d6`, Spent `#eb6834`) and layout follow the design canvas's
Overview artboard, except that the tab is placed after Income and the third tile is called
"Saved".
