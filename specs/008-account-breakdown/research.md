# Research: Spending by account

Every decision mirrors what By category already does (features 003, 006, 007) unless a reason is
given. Nothing here was left open.

## R-001 Where account colours live

- **Decision**: a stored `accounts.color` column (`#rrggbb`, NOT NULL), added by Flyway V6, set
  by the backend when an account is created and never changed afterwards (the entity field is not
  updatable, as for `categories.color`).
- **Rationale**: the same reasoning as feature 007 R-002. A colour derived at read time (by rank
  or by a hash of the name) would change when the account is renamed or another account is added,
  which breaks FR-020. The backend owns data (Principle II), so the frontend only displays it.
- **Alternatives considered**: hashing the account id into the palette on the frontend (no
  migration, but two accounts can collide while colours are still free, breaking FR-021, and it
  puts a rule in the frontend); a user-chosen colour (out of scope, spec Assumptions).

## R-002 Which palette and which colour a new account gets

- **Decision**: reuse `CategoryPalette` unchanged. A new account gets
  `CategoryPalette.next(counts)`, where `counts` are the colours of the caller's accounts only.
  So the first account is `#e11d48`, then the other six spare colours, then the defaults' colours
  in default-category order, then the cycle repeats from the least-used colour.
- **Rationale**: FR-023 asks for the same palette, and `next` already implements "least used,
  ties in a fixed order" (007 R-004). Counting only accounts keeps account and category colours
  independent (spec Key Entities). Starting with the spare colours means a user's first seven
  accounts are not in the same colours as the busiest default categories, which reads better when
  both cards are on screen.
- **Alternatives considered**: a second palette for accounts (more colours to check for 3:1
  contrast and distinctness for no user benefit); renaming `CategoryPalette` to `Palette`
  (cosmetic churn across files and tests; the KDoc is updated instead).

## R-003 Back-filling existing accounts

- **Decision**: V6 adds the column with a neutral default (`#8a908b`, outside the palette, the
  same trick as V5 so the previous build keeps inserting during a rolling deploy), then sets each
  existing account's colour by creation order per user: the n-th account (by `created_at, id`)
  gets position `(n − 1) mod 24` of the tie order (7 spare colours, then the 17 default ones). A
  `CHECK (color ~ '^#[0-9a-f]{6}$')` constraint is added last.
- **Rationale**: this is exactly the sequence `next` produces when accounts are created one by one
  into an empty set, so back-filled and newly created accounts follow one rule (FR-022). The
  default is not dropped, matching V5.
- **Alternatives considered**: assigning in alphabetical order (would differ from what creation
  order produces); leaving old accounts grey (breaks FR-022 and SC-004).

## R-004 Totals per account from the server

- **Decision**: a new read-only endpoint `GET /api/v1/account-totals?from=&to=`, the twin of
  `GET /api/v1/category-totals`: same parameters and range rules (ordered, at most 366 days),
  same shape with `account` in place of `category`, only accounts with spending, largest first,
  ties by name. New schemas `AccountTotals` and `AccountTotal` (with `color`). A new
  `TotalsQueries.byAccount` joins `accounts` the way `byCategory` joins `categories`; sums are in
  Postgres on `numeric`, so FR-007 holds to the cent. Not paged, for the same reason as category
  totals (bounded by the user's account count; 003 plan Complexity Tracking).
- **Rationale**: one small, additive endpoint; the screen gets exactly what it renders. The
  existing `ix_spendings_user_date` index serves the date filter, as for categories, so SC-002 is
  met; `TotalsPerformanceIT` gets the same 10,000-spending check for the new endpoint.
- **Alternatives considered**: a `groupBy=category|account` parameter on one `/spending-totals`
  endpoint (a different response shape per value is awkward in OpenAPI and would mean
  deprecating `/category-totals`); summing on the frontend from the month's spendings (puts
  business logic in the frontend, Principle II, and the list endpoint is paged).

## R-005 `Account.color` in the API

- **Decision**: add a required `color` field to the `Account` response (list, get, create,
  rename) and to `AccountTotal`. Requests are unchanged.
- **Rationale**: additive response fields are backward compatible (Principle VII), and it mirrors
  `Category.color`. The breakdown only needs `AccountTotal.color`, but exposing it on `Account`
  keeps the two resources consistent and lets Settings show a swatch later without an API change.
- **Alternatives considered**: only on `AccountTotal` (inconsistent with categories).

## R-006 One breakdown component on the desktop

- **Decision**: generalise the desktop By category card instead of copying it. `ByCategoryCard`,
  `CategoryTable` and `CategoryPie` become `BreakdownCard`, `BreakdownTable` and `BreakdownPie`,
  configured by a `kind` (`'category' | 'account'`) that sets the heading ("By category" / "By
  account"), the first column header, the chart summary wording, the swatch test id and the
  localStorage key (`hlb.byCategoryView` stays as it is; `hlb.byAccountView` is new, so each card
  keeps its own Table / Pie choice, FR-002). `MonthSection` renders a second card below the first
  inside the existing summary `aside`, so it already moves below the table on narrow windows
  (FR-010).
- **Rationale**: "same as by category" is literally the same component with different labels;
  one component means a later change to one breakdown cannot silently miss the other.
- **Alternatives considered**: duplicating the three files for accounts (two copies to keep in
  step); one card with a Category / Account switch (rejected by the spec's FR-010).

## R-007 Phone: a third view and its own chart choice

- **Decision**: the phone view state gains `'by-account'`, and the switch in `MonthYearPicker`
  gets a third option, "By account". `CategoryBreakdown` becomes a `Breakdown` component with the
  same `kind` setting as R-006. The phone UI state gets a second chart choice, `accountChart`,
  beside the existing `chart`, so By category and By account each keep their own Table / Pie
  (FR-002), and both survive month changes because the provider sits above the `<Suspense>`
  (006 data-model). `topCategories` in `app/lib/phone.ts` is renamed `topSlices`; it already takes
  any `{id, name, cents, color}` items.
- **Rationale**: the phone breakdown differs from the desktop one only in layout (top 6), so it
  gets the same treatment as the desktop.
- **Alternatives considered**: one shared chart choice for both views (simpler state, but
  contradicts FR-002 and surprises a user who wants Table for one and Pie for the other).

## R-008 Three options in the phone switch

- **Decision**: keep `Segmented` (equal-width columns, 44px min height) and let the label wrap to
  two lines inside the button if it must; use `px-2` instead of `px-3` when there are three or
  more options. At 360px wide each option is about 101px, which fits "By category" on one line;
  at 320px it wraps to two lines and stays 44px tall. No sideways scrolling (FR-015).
- **Rationale**: smallest change; no new control.
- **Alternatives considered**: shortening labels to "Categories" / "Accounts" (diverges from the
  desktop headings); a dropdown (one more tap, hides the options).

## R-009 Loading the data

- **Decision**: `loadSpendingsPage` fetches `getAccountTotals(ym)` in its second parallel round,
  next to `getCategoryTotals(ym)`, and both the desktop and phone sections read it. After any add,
  edit, delete or import, the existing server actions call `refresh()`, which re-renders the page
  with the new totals (FR-008); nothing else is needed.
- **Rationale**: same pattern as category totals; one extra parallel call does not lengthen the
  page load.
- **Alternatives considered**: fetching only when By account is opened on the phone (client-side
  fetch; the frontend reads the API from the server only).

## R-010 Delivery order

- **Decision**: backend first (V6, colour on create, `Account.color`, `/account-totals`), deployed
  on its own; then the frontend. Both changes are additive, so the frontend on the old build and
  the backend on the new one work together in either order of rollout.
- **Rationale**: the frontend's types are generated from the backend's exported contract.
