# Research: Fixed category colours, a working pie chart, and categories and accounts managed only in Settings

Feature 007. Each decision below resolves a question the plan's Technical Context left open.

## R-001 Why most pie slices have no colour

**Finding**: Both donuts colour slices with CSS custom properties built at run time:
`var(--color-cat-${i + 1})` in `app/ui/spendings/category-colors.ts` (desktop) and
`app/lib/phone.ts` (phone). The 12 variables are declared in the Tailwind v4 `@theme` block of
`app/globals.css`. Tailwind v4 emits a theme variable only when it finds it used in the sources
it scans, and it cannot see a name assembled from a template string. Variables that happen to be
referenced elsewhere survive; the rest are dropped, so `fill="var(--color-cat-5)"` points at
nothing and the slice renders without a colour. This matches the symptom: the first slice (the
largest, usually Rent) and "some random" others are coloured.

**Decision**: Stop colouring slices through theme variables. Each category's colour arrives from
the API as a hex string (R-003) and is set directly as the slice `fill`, the legend swatch
`background` and the table swatch. The `--color-cat-*` tokens, `category-colors.ts` and the
phone's `CAT_COLORS` are removed. The phone's grey "Everything else" slice uses a literal hex
value for the same reason.

**Alternatives considered**:
- `@theme static` or listing the variables in a safelist: fixes the symptom, but colours would
  still follow rank, not category, so it does not meet FR-001/FR-002.
- Tailwind classes such as `fill-cat-5`: same rank problem, and dynamic class names hit the same
  scanning limit.

## R-002 Where category colours live

**Decision**: The backend owns them. A `color` column on `categories` holds a `#rrggbb` value.
The palette and the assignment rule live in the backend (`model/CategoryPalette.kt`). The
frontend only displays what the API returns.

**Rationale**: Constitution II: the backend owns data and business rules. "Defaults have fixed
colours, a new category gets the least-used colour" is a business rule, and it must hold for
every client and every device (SC-002). Storing the hex, not a palette index, means a later
palette change never silently recolours existing categories.

**Alternatives considered**:
- Frontend-only mapping (hash of the category id into the palette): no storage, but cannot
  guarantee "different from the user's other categories" (FR-002), and the rule would live in the
  UI, against Constitution II.
- Storing a palette index: smaller, but ties stored data to the palette's order forever.

## R-003 The palette

**Decision**: 24 colours. The first 17 belong to the default categories, in the order of
`DefaultCategories`; the last 7 are the spare colours for custom categories. Each is at least
3.3:1 against the white card (WCAG non-text contrast is 3:1), and the closest pair still differs
by ΔE ≈ 12.6 (CIE76); every other pair differs more.

| # | Colour | Given to | # | Colour | Given to |
|---|---|---|---|---|---|
| 1 | `#1e5b45` | Rent | 13 | `#64748b` | Other |
| 2 | `#c2410c` | Groceries | 14 | `#16a34a` | Sport |
| 3 | `#1d4ed8` | Public Transport | 15 | `#c026d3` | Entertainment |
| 4 | `#a16207` | Car | 16 | `#7c2d12` | Pet |
| 5 | `#7e22ce` | Household | 17 | `#8a6d00` | Taxi |
| 6 | `#0f766e` | Health | 18 | `#e11d48` | spare |
| 7 | `#be185d` | Beauty | 19 | `#0891b2` | spare |
| 8 | `#4d7c0f` | Clothes | 20 | `#a855f7` | spare |
| 9 | `#b91c1c` | Education | 21 | `#365314` | spare |
| 10 | `#0369a1` | Travel | 22 | `#1e3a8a` | spare |
| 11 | `#57534e` | Bills | 23 | `#701a75` | spare |
| 12 | `#4338ca` | Subscriptions | 24 | `#ea580c` | spare |

Colours 1–10 are the current `--color-cat-1..10` values, so the most common categories keep a
look the user already knows. A backend unit test checks that there are 24 distinct values, that
each is at least 3:1 against `#ffffff`, and that the 17 defaults get 17 different colours.

**Alternatives considered**: 30+ colours. Beyond about 24, colours at 3:1 against white become
hard to tell apart; past the spare 7, the least-used rule (R-004) plus the slice gap, legend and
hover label keep slices identifiable.

## R-004 Assigning a colour to a new category

**Decision**: When a category is created (only in Settings, `POST /api/v1/categories`), the
service counts how many of the caller's categories use each palette colour and picks the
least-used one. Ties go to the first colour in the order spare colours first (18…24), then
1…17. Renaming keeps the colour; deleting frees it simply because it is no longer counted.

With the 17 defaults always present (they cannot be deleted), this gives: the first 7 custom
categories get colours 18…24; after that the order cycles 18…24, 1…17, 18…24, … as counts even
out.

No lock or unique constraint: two categories created at the same instant may get the same
colour, which the spec allows (colours may repeat).

**Alternatives considered**: round-robin by count of custom categories: simpler, but after a
deletion it would hand out a colour still in use while another is free.

## R-005 Back-filling existing rows

**Decision**: Flyway `V5__add_category_colors.sql`:
1. Add `color varchar(7)` (nullable).
2. Defaults: set by name from the R-003 table (a `VALUES` list joined on `name` and
   `is_default = true`).
3. Custom categories: number each user's custom categories by `created_at, id` (`rn` from 1) and
   apply the R-004 sequence in closed form: `rn ≤ 7` → colour `17 + rn`; otherwise colour
   position `((rn − 8) mod 24)` in the order 18…24, 1…17. This equals running R-004 once per
   category in creation order, since nothing has been deleted from the assignment's point of
   view.
4. `SET NOT NULL` and `CHECK (color ~ '^#[0-9a-f]{6}$')`.

A migration test (Testcontainers) seeds a user with defaults and 10 custom categories on V4,
migrates, and checks the colours.

**Rationale**: SQL keeps it inside the Flyway history (Constitution V) without a Java-based
migration. Users who have not been seeded yet get colours when they are seeded (R-006).

## R-006 Seeding defaults with colours

**Decision**: `DefaultCategories.NAMES` becomes `DefaultCategories.ALL: List<DefaultCategory>`
(name and colour, from `CategoryPalette`). `CurrentUserRepository.insertDefaultCategories`
writes both.

## R-007 Exposing colours in the API

**Decision**: Two additive response fields in v1:
- `Category.color` (required, read-only, `#rrggbb`) on every category response.
- `CategoryTotal.color` (required, `#rrggbb`) on `GET /api/v1/category-totals` items, next to
  the existing `category` ref, so the donut needs no second request.

`Ref` (used by spendings) is not changed.

**Compatibility**: backward compatible (new response fields only). The frontend regenerates
`app/lib/api-schema.d.ts`.

## R-008 Refusing unknown names instead of creating them

**Decision**: Keep the request shapes. Change what a name means:
- `SpendingRequest.categoryName` / `accountName` (create and replace a spending): matched against
  the caller's items ignoring case and surrounding spaces; when none matches, a `422` with a
  violation `{pointer: "#/categoryName", code: "NotFound", message: "is not one of your
  categories"}` (accounts: `#/accountName`, "is not one of your accounts"). This mirrors the
  existing `NotFound` violation for an unknown `categoryId`.
- `SpendingImportRow.categoryName` / `accountName`: the same per row, with pointers
  `#/spendings/{i}/categoryName` and `#/spendings/{i}/accountName`. All rows' problems are
  reported at once; nothing is written (FR-027).
- `NamedItemResolver` stops creating: `byName` and `resolveNames` look names up (one query per
  kind for an import: all of the caller's names, keyed by `key()`), and add violations for misses.
  `NamedItemInserts.findOrCreateCategory/Account` and their tests are removed.
- The lookup happens in the same transaction as the insert. A category or account still in use
  cannot be deleted (feature 002), and the spending insert's foreign key takes a share lock on the
  referenced row, so a concurrent delete cannot slip between the check and the insert.

**Compatibility**: no schema, field, status code or problem `type` changes. `422` with
`errors[]` is already a documented response of both endpoints, and `NotFound` is already a
documented violation code. What changes is a business rule: input that used to create an item is
now refused. The frontend is the only client and is updated in the same feature. See
Complexity Tracking in plan.md for why this is not a v2.

**Alternatives considered**:
- Remove `categoryName`/`accountName` and require ids: a breaking change (Constitution VII,
  needs `/api/v2`) for no user benefit; matching by name still helps the import, whose rows
  carry names.
- A new `/api/v2/spendings` and `/api/v2/spending-imports`: doubles two endpoints and their tests
  for one rule change in a single-client, single-user app (Constitution VI).

## R-009 Frontend validation of names

**Decision**: `draftProblems(draft, today)` gains a third argument, `lookups: { categories,
accounts }` (the user's names, already loaded wherever a draft is edited: entry row, phone form,
import review). A non-empty category that matches no item (same `normaliseName` rule as the CSV
reader) gets `MESSAGES.unknownCategory` = "Not one of your categories"; accounts get
`MESSAGES.unknownAccount` = "Not one of your accounts". Empty or too long keep their current
messages. Because the entry row, phone form, review table, phone Fix row and the add/save Server
Actions all call `draftProblems`, one change covers every surface.

On save, the Server Actions map a `422` `NotFound` violation on `categoryName`/`accountName`
(and the per-row pointers for imports) to the same field and message, so a name removed in
another tab shows exactly like a local miss (FR-027).

When the typed value matches an item, the action sends its `categoryId`/`accountId`; names are
sent only from the import. `MESSAGES.categoryName`/`accountName` ("Type a name …") stay for
Settings.

## R-010 Suggestions only from existing items

**Decision**: The desktop combobox and the phone suggest input already list only existing items.
The combobox's `createHint` prop (shows "Create “x”" when the text names no option, used for
categories and accounts) is removed with its callers' `createHint` flags and tests. The phone
suggest input gets the same check. Free typing stays (it is needed to filter), and an unmatched
value is caught by R-009. Accepting a past merchant fills
category and account from the merchant API, which returns names of items still referenced by
spendings and so still existing.

## R-011 A user with no accounts

**Decision**: The desktop entry row and the phone Add screen show, in place of the account field,
"Add an account in Settings first." with a "Open Settings" link/button (desktop: opens the
Settings window on Accounts; phone: opens phone Settings on the Accounts tab). The save button is
disabled. The accounts panel's empty text drops "or type a new account name when you record a
spending" and becomes "You have no accounts yet. Add one to start recording spendings." The
import review shows every account field invalid ("Not one of your accounts"), which already
blocks adding.

## R-012 Hover, focus and tap label on the donut

**Decision**: A small in-component tooltip, no library:
- The `<svg>` changes from `role="img"` (which hides its children from screen readers) to
  `role="group"`, labelled with the existing summary. Each slice `<path>` gets `tabIndex={0}`,
  `role="img"` and `aria-label="Rent: $1,850.00, 42%"`, so keyboard and screen-reader users can
  reach every slice.
- State `active: number | null` set on `pointerenter`/`focus`, cleared on `pointerleave` of the
  chart and `blur`. On touch (`pointerType === 'touch'`), a tap toggles the slice; a tap outside
  the chart clears it (`pointerdown` listener on the document while a label is shown).
- The label is an absolutely positioned `<div role="status" aria-live="polite">` over the chart's
  centre (replacing the "Total" text while active), showing "Rent", "$1,850.00 · 42%". Centre
  placement never overflows the card and needs no positioning maths. The native `<title>` is
  removed (it would show a second, delayed tooltip).
- The active slice is emphasised by dimming the others to 35% opacity and drawing a 2px outline in
  `--color-ink` on the active one.
- Shared by desktop and phone: a `DonutChart` component in `app/ui/charts/donut-chart.tsx` takes
  `slices: { key, label, amountText, shareText, color }[]` and is used by `category-pie.tsx` and
  `phone/category-breakdown.tsx`.

**Alternatives considered**: a chart library (Recharts, visx): a new dependency for one donut
(Constitution VI). A tooltip that follows the pointer: needs viewport clamping and does not work
for focus or tap.

## R-013 Table view swatch

**Decision**: `category-table.tsx` shows a 12px square swatch in the category's colour before
each name (decorative, `aria-hidden`), using the `color` from the totals response.

## R-014 Delivery order

**Decision**: Backend first (V5 migration, colour fields, name rule), deployed; then the frontend
(regenerated types, colours, label, validation). Between the two deploys, the old frontend
still works: it ignores the new fields, and a typed new name now gets the server's "is not one of
your categories" message on save instead of creating the item. That window is minutes and only
affects a typo path.
