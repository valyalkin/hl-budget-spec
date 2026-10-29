# Data Model: Spending Screens (Web App)

No tables, columns or migrations change. The stored entities are feature 002's (`accounts`,
`categories`, `spendings`; see
[../002-spending-tracking/data-model.md](../002-spending-tracking/data-model.md)). This feature
adds **derived read models** on the backend, one **write rule**, and the **UI state** the screens
hold.

## Derived read models (backend, computed per request)

### MonthTotal (`GET /monthly-totals`)

| Field | Type | Rule |
|---|---|---|
| `month` | `YYYY-MM` | Every month of the requested year, January to December |
| `total` | MoneyTotal | `sum(amount)` of the caller's spendings dated in that month; `0.00` when none |
| `spendingCount` | int64 | Number of those spendings |

Plus `year` and `today` (today in the app time zone, `HlBudgetProperties.zoneId`).
Query: `GROUP BY date_trunc('month', spending_date)` with `user_id = ? AND spending_date BETWEEN
Jan 1 AND Dec 31`, served by `ix_spendings_user_date`; missing months filled with zero in the
service.

### CategoryTotal (`GET /category-totals`)

| Field | Type | Rule |
|---|---|---|
| `category` | Ref | Category id and current name |
| `total` | MoneyTotal | Sum in range; always > 0 (only categories with spending appear) |
| `spendingCount` | int64 | ≥ 1 |

Order: `total` desc, then `lower(name)`. Envelope adds `from`, `to`, range `total`,
`spendingCount`. Validation: both dates required, `from ≤ to`, range ≤ 366 days (422 otherwise).

### Merchant (`GET /merchants`)

| Field | Type | Rule |
|---|---|---|
| key (not returned) | text | `lower(trim(merchant))` |
| `name` | text | Merchant spelling on the most recent spending in the group |
| `spendingCount` | int64 | Spendings in the group |
| `lastUsedOn` | date | Date of the most recent spending |
| `lastCategory`, `lastAccount` | Ref | Category and account of the most recent spending |

"Most recent" = latest `spending_date`, then latest `created_at`, then `id` (the list's
tie-break). Order: `spendingCount` desc, then `lower(name)`. `limit` 1–1000 (default 500),
`truncated` true when more groups exist.

## Write rule changes (backend)

### Spending create / update: category and account by id or by name

- Exactly one of `categoryId` / `categoryName`, and exactly one of `accountId` / `accountName`;
  otherwise 422 with pointer `/categoryId` or `/accountId`.
- A name is trimmed and validated like `NameRequest` (non-empty, ≤ 50). It matches the caller's
  item with the same `lower(name)`; if none, the item is inserted (categories as
  `is_default = false`). A concurrent insert of the same name is caught on the unique index and
  re-read, so both requests end up using one item.
- Resolution, creation and the spending write share one transaction: a refused spending leaves
  no new item behind.
- The idempotency fingerprint (002 R-007) is computed over the resolved ids, so a retry that
  replays by name matches the first request.

### Account delete: last-account rule

Refusals, in order: not found (404) → used by spendings (409 `in-use`) → the caller's only
account (422 `last-account`). The caller's account rows are locked for the check, so two
concurrent deletes cannot both pass and leave zero accounts.

## UI state (frontend, not persisted)

| State | Lives in | Values |
|---|---|---|
| Selected month | URL `?month=YYYY-MM` | Defaults to `today`'s month; future or invalid → current month |
| Shown year on the month line | Derived from selected month; arrows navigate to `?month=<year>-<same or last allowed month>` | Next arrow disabled at `today`'s year |
| Open row | `spendings-table` client state | `null` \| `{ mode: 'add' }` \| `{ mode: 'edit', spendingId }`; at most one (FR-033) |
| Row draft | `entry-row` client state | `date`, `merchant`, `category` (`{ id }` picked or `{ name }` typed), `account` (same), `amount` (text), `idempotencyKey` (UUID, add only), `error` |
| Autofill suggestion | `combobox` client state | typed text, suggested completion, active option index |
| By category view | `localStorage` (`hlb.byCategoryView`), default `table` | `table` \| `pie`; kept across month changes (FR-070) |
| Settings | `settings-dialog` client state | `open`, `section: 'categories' \| 'accounts'`, add-field text, message |

### Row draft lifecycle

```text
closed ──(+ or edit)──▶ editing ──(save valid)──▶ saving ──(ok)──▶ closed, navigate to month
                          │  ▲                       │
                          │  └──────(error)──────────┘   draft kept, message shown
                          └──(X / Escape)──▶ closed, nothing saved
```

### Draft validation (local, mirrors the server; first failure wins, FR-037)

1. `date` present and ≤ `today` → "Pick a date." / "Pick a date that is today or earlier."
2. `merchant` non-blank after trim, ≤ 255 → "Add a merchant name."
3. `category` picked or non-blank typed name, ≤ 50 → "Choose or type a category."
4. `account` the same → "Choose or type an account."
5. `amount` matches `^\d{1,8}([.,]\d{1,2})?$` and > 0 → "Enter an amount greater than zero."

A typed name equal (ignoring case and surrounding spaces) to an existing item is sent as that
item's id.
