# Data Model: Feature 007

Changes only. Everything not listed is as in features 002–004.

## Category (table `categories`) — changed

| Field | Type | Rule |
|---|---|---|
| `color` | `varchar(7)`, NOT NULL, default `'#8a908b'` | New. `#rrggbb`, lower case (`CHECK (color ~ '^#[0-9a-f]{6}$')`). Set once when the category is created; never changed by rename. The default (a grey outside the palette) only covers the previous build inserting categories during a rolling deploy; the app always sets the colour. |

**Assignment** (research R-004):
- Default category: its fixed colour from the palette (R-003), by name.
- Custom category (created in Settings): the palette colour used by the fewest of the user's
  categories; ties go to the first in the order 18…24, 1…17.

**Migration** `V5__add_category_colors.sql` (R-005): add the column with the grey default, back-fill defaults by
name and custom categories by creation order, then add the check. Applied migrations V1–V4
are untouched.

**Who can create a category**: only `POST /api/v1/categories` (Settings). Spending create,
spending replace and spending import no longer create categories.

## Account (table `accounts`) — rule changed, no schema change

Created only by `POST /api/v1/accounts` (Settings). Spending create, replace and import no longer
create accounts.

## CategoryPalette (code, `model/CategoryPalette.kt`) — new

An ordered list of 24 hex colours (R-003). Positions 1–17 are the defaults in
`DefaultCategories` order; 18–24 are spare. Not stored in the database; stored categories keep
their hex even if the palette changes later.

## DefaultCategories (code) — changed

`NAMES: List<String>` becomes `ALL: List<DefaultCategory(name, color)>`; seeding writes both.

## Name matching (unchanged rule, new outcome)

A category or account name matches an item when `lower(trim(name))` equals the item's
`lower(name)`. No match:
- spending create/replace by name → `422`, violation `#/categoryName` or `#/accountName`, code
  `NotFound`;
- import → `422`, violation `#/spendings/{i}/categoryName` or `…/accountName`, code `NotFound`,
  for every failing row; nothing written.

## Frontend shapes — changed

- `CategoryItem` (By category card) gains `color: string`, from `CategoryTotal.color`.
- `NamedItem` lists for categories/accounts (already loaded) are passed to `draftProblems` as
  `lookups`.
- `DraftProblems.category` / `.account` gain the messages "Not one of your categories" / "Not one
  of your accounts".
- `DonutSlice { key, label, amountText, shareText, color }` — input of the shared `DonutChart`.

## State: donut label

| State | Shown |
|---|---|
| `active = null` | Centre shows "Total" and the month total; all slices full opacity |
| `active = i` | Centre shows slice *i*'s name and "amount · share"; other slices dimmed; slice *i* outlined |

Transitions: pointer enters slice / slice gets focus / tap on slice → `active = i`; pointer leaves
chart / blur out of chart / tap outside chart / tap the active slice again → `null`.
