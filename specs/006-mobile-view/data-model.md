# Data Model: Mobile View

**Feature**: [spec.md](./spec.md) | **Date**: 2026-10-31

There is **no new stored data and no server change**. Spendings, categories, accounts and
merchants are the server entities from features 002–004, read through the existing loaders
(`app/lib/data.ts`) and changed through the existing Server Actions (`app/lib/actions.ts`).

This document describes the client-side state the phone layout holds while it is open. None of
it is persisted. A reload starts from the Spendings list (FR-103).

---

## Phone page data (read-only, from the server)

What the phone section receives per request. It is the same `SpendingsPageData` the desktop
uses (`loadSpendingsPage`, now wrapped in React `cache` so the two layouts share one load), plus
one field.

| Field | Source | Used for |
|---|---|---|
| `ym`, `today` | `loadSpendingsPage` | Selected month and year, future-month and future-day limits |
| `totals` | `loadSpendingsPage` | Month total and count (FR-016) |
| `spendings` | `loadSpendingsPage` | The list (FR-018), newest first |
| `categoryTotals` | `loadSpendingsPage` | Table and Pie (FR-031, FR-032) |
| `categories` (with `count`, `isDefault`) | `loadSpendingsPage` | Category suggestions, Settings |
| `accounts` (with `count`) | `loadSpendingsPage` | Account suggestions, Settings usage text (FR-082) |
| `merchants` (`lastCategory`, `lastAccount`) | `loadSpendingsPage` | Merchant suggestions and fill (FR-046) |
| `earliestYear` | **new** `getEarliestSpendingYear()` (R-004) | Year dropdown range (FR-013) |

## Phone UI state (`PhoneUiProvider`, outside `<Suspense>`)

| Field | Type | Rules |
|---|---|---|
| `view` | `'spendings' \| 'by-category'` | Starts `'spendings'`; kept across month changes |
| `chart` | `'table' \| 'pie'` | Starts `'table'`; kept across month changes (FR-030) |
| `screen` | `'list' \| 'edit' \| 'settings' \| 'import' \| 'review' \| 'fix-row'` | One at a time; `'list'` on load |
| `sheet` | `'none' \| 'menu' \| 'confirm-delete'` | `menu` only over `list`; `confirm-delete` only over `edit` (existing spending) |
| `historyDepth` | number (0–3) | Count of history entries this layout pushed (R-006) |
| `notice` | string or empty | "Deleted [merchant] · [amount]" or "Added N spendings from [file]"; cleared on the next action |
| `settingsTab` | `'categories' \| 'accounts'` | Starts `'categories'` each time Settings opens |
| `import` | Import session or null | See below; null outside the import screens |

### Screen transitions

```text
list ──tap row──────────▶ edit (existing)  ──Save changes ok──▶ list (month of spending)
list ──Add spending─────▶ edit (new)       ──Add spending ok──▶ list (month of spending)
edit (existing) ──Delete spending──▶ sheet confirm-delete ──Delete ok──▶ list + notice "Deleted …"
                                                          ──Cancel / backdrop / Back──▶ edit
list ──user icon──▶ sheet menu ──Settings──▶ settings ──X / Back──▶ list
                               ──Import CSV──▶ import ──file ok──▶ review
                               ──Sign out──▶ /auth/logout → "You're signed out"
review ──tap row──▶ fix-row ──Save row (valid)──▶ review
                            ──Remove row──▶ review (row removed)
                            ──X / Back──▶ review (unchanged)
review ──back button / Back──▶ import (rows discarded)
review ──Add N spendings ok──▶ list (latest imported month) + notice "Added …"
any screen ──X / Back──▶ previous screen, nothing saved
width ≥ 640px ──▶ desktop layout; all phone screens and sheets closed (Edge Cases)
```

## Spending form (Edit / Add / Fix row)

Reuses the desktop's `Draft` type (`app/lib/draft.ts`): `date`, `merchant`, `category`,
`account`, `amount`, all strings while typing.

| Field | Initial value: add | Initial value: edit | Initial value: fix row |
|---|---|---|---|
| `amount` | empty | spending's amount | row's amount (may be empty) |
| `date` | `defaultDateFor(ym, today)`: today in the current month, else the 1st (FR-043) | spending's date | row's date (may be empty) |
| `merchant`, `category`, `account` | empty | spending's values | row's values (may be empty) |

Extra form state: `mode` (`'add' | 'edit' | 'fix-row'`), `spendingId` (edit), `rowId` (fix
row), `calendarOpen`, `calendarMonth`, `problem` (first `DraftProblem` in phone order, R-010),
`pending` (an action is running), `serverMessage`.

**Validation**: `draftProblems(draft, today)`, first problem in the order `amount, date,
merchant, category, account`. Save is blocked while any problem exists, for all three modes
(clarified for fix row).

**Mode labels** (FR-041, FR-060, FR-094):

| Mode | Title | Bottom button | Delete button | Delete confirmation |
|---|---|---|---|---|
| add | New spending | Add spending | none | — |
| edit | Edit spending | Save changes | Delete spending (red outline) | Bottom sheet |
| fix-row | Fix row | Save row | Remove row | None (removes at once) |

## Import session (phone)

| Field | Type | Rules |
|---|---|---|
| `fileName` | string | From the picked file; shown in the review header and the "Added …" notice |
| `rows` | `ImportRow[]` (004's type) | File order; produced by `readImport` / `rowsFromCsv` |
| `idempotencyKey` | UUID | Created when the review opens; reused on retry (004 FR-034) |
| `error` | string or empty | Pick-step error (not CSV, empty, unreadable, too large, no rows) |
| `pending` | boolean | Add in progress; disables the button |

**Derived per row**: `problems = draftProblems(row.draft, today)`; `needsAttention` = any
problem. The display texts are:

| Part | Shown when it has a problem | Shown otherwise |
|---|---|---|
| date | "No date" (empty or unreadable); the date itself in amber (after today) | "Oct 27" |
| merchant | "Merchant missing" | merchant name |
| category | "No category" | category name |
| account | "No account" | account name |
| amount | "No amount" | "$12.40" |

**Derived for the screen**:
- banner: "N rows need attention" when any row needs attention, otherwise "All rows look good";
- add button: disabled with the label "Add spendings" while any row needs attention or no rows
  remain; otherwise "Add N spendings" (or "Add 1 spending").

## Category breakdown slice (Pie)

From `topCategories(categoryTotals.items, 6)` (R-013):

| Field | Rules |
|---|---|
| `name` | Category name, or "Everything else" (only when there are more than 6 categories) |
| `cents` | Category total; "Everything else" = sum of the rest |
| `share` | `sharePercent(cents, monthTotalCents)` |
| `color` | `--color-cat-1` … `--color-cat-6` by rank; "Everything else" uses `--color-muted` |

The Table view uses every category, unchanged, with a final Total row at 100%.

## Year options

`years = [today.year, today.year − 1, …, min(earliestYear, ym.year)]`, newest first. There is
always at least the current year.

Month options: January–December for the selected year; in the current year, months after
`today`'s month are `disabled`.
