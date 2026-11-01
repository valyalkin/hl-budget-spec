# Data Model: Feature 008

Only what changes. Everything else is as in features 002, 003 and 007.

## Account (table `accounts`) — changed

| Column | Type | Rule |
|---|---|---|
| `color` | `varchar(7)` NOT NULL, default `#8a908b` | `#rrggbb`, lower-case hex (`CHECK color ~ '^#[0-9a-f]{6}$'`). Set once on create, never updated (entity field `updatable = false`); renaming keeps it (FR-020) |

- **New account**: `color = CategoryPalette.next(colour counts of the caller's accounts)`
  (research R-002). The count query mirrors `CategoryRepository.colorCounts`, scoped to
  `user_id`.
- **Existing accounts** (V6): the n-th account of each user by `created_at, id` gets position
  `(n − 1) mod 24` of the tie order: spare `#e11d48 #0891b2 #a855f7 #365314 #1e3a8a #701a75
  #ea580c`, then the 17 default category colours in `CategoryPalette` order (research R-003).
- **Removed account**: unchanged rules (only when unused and not the last one); its colour is
  simply no longer counted, so it is free for the next account.
- The default value exists only for rolling deploys; the app always writes a palette colour.

Model `Account` gains `color: String`; `AccountRow` (the counted read) selects `a.color`.

## AccountTotal (derived, not stored) — new

For a date range: one row per account with at least one spending in it.

| Field | Meaning |
|---|---|
| `account` | `Ref` (id, name) |
| `total` | sum of the spendings' amounts, exact (`numeric`) |
| `spendingCount` | number of spendings, ≥ 1 |
| `color` | the account's `color` |

`AccountTotals` = `from`, `to`, `total` (sum of the items), `spendingCount`, `items` ordered by
`total DESC, lower(name), id`. Range rules as for category totals: `from ≤ to`, at most 366 days.

Query (`TotalsQueries.byAccount`): `spendings s JOIN accounts a ON a.id = s.account_id AND
a.user_id = s.user_id WHERE s.user_id = :userId AND s.spending_date BETWEEN :from AND :to GROUP BY
a.id, a.name, a.color`.

## CategoryPalette (code) — doc only

KDoc now says the palette serves accounts as well; the list and `next` are unchanged.

## Frontend shapes — changed

- `SpendingsPageData` gains `accountTotals: AccountTotals` (generated type).
- `BreakdownItem` = `{ id, name, cents, color }` (today's `CategoryItem`, renamed), used for both
  kinds.
- `BreakdownKind = 'category' | 'account'` sets heading, column header, summary text, swatch test
  id (`category-swatch` / `account-swatch`) and the desktop storage key.
- Phone `PhoneData` gains `accountTotals: { totalCents, items: BreakdownItem[] }`.
- Phone UI state: `view: 'spendings' | 'by-category' | 'by-account'`; new
  `accountChart: 'table' | 'pie'` (default `'table'`) beside `chart`, with `setAccountChart`.
- `topCategories` → `topSlices` (same behaviour: top 6 + "Everything else" in `#8a908b`).

## State: chosen view per breakdown

| Where | Category | Account | Kept across |
|---|---|---|---|
| Desktop | localStorage `hlb.byCategoryView` | localStorage `hlb.byAccountView` | month changes and reloads |
| Phone | `PhoneUi.chart` | `PhoneUi.accountChart` | month changes (provider above `<Suspense>`) |
