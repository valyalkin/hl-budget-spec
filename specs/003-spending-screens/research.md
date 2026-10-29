# Research: Spending Screens (Web App)

Phase 0 decisions for [plan.md](./plan.md). Each entry: **Decision**, **Rationale**,
**Alternatives considered**. Spec references are to [spec.md](./spec.md).

---

## R-001 — Routes and the signed-out screen

**Decision**:

- `/spendings` is the Spendings screen. It joins `PROTECTED_PREFIXES` in `proxy.ts`, so a
  request without a session never renders it.
- `/` becomes the signed-out screen from the design: the new app header (with Sign in) and
  "You're signed out / Sign in to see and record your spendings." When a session exists, `/`
  redirects to `/spendings`. Sign-out (`/auth/logout`) already returns to `/`.
- `/home` (001's placeholder) permanently redirects to `/spendings` in `next.config.ts`
  `redirects()`. The 001 profile card and `app/home/` are removed; `ensureMyProfile()` moves into
  the Spendings data load, so the user record is still created or refreshed on every visit.
- The selected month lives in the URL: `/spendings?month=2026-10`. A missing or invalid value
  falls back to the current month (FR-012). A future month is clamped to the current one.
- Both routes carry `robots: { index: false, follow: false }` (FR-093). `/` stays in neither the
  sitemap nor the robots allow list for authed routes.

**Rationale**: The proxy gate keeps the 001 security model (no data render without a session)
instead of a page that branches on the session. The design's signed-out message is exactly what
a visitor at `/` should see. Month-in-URL survives reload and the back button, and lets the
server render the month directly (the skill's "state in the URL" rule).

**Alternatives considered**:
- *One route `/` that shows either state*: renders personal data on a route the proxy cannot
  gate, and makes every request dynamic. Rejected.
- *Keep `/home`*: the name no longer fits; a redirect keeps old bookmarks and 001's e2e journey
  working.
- *Month in client state*: lost on reload, and forces client-side fetching.

---

## R-002 — Where totals come from: two read-only summary resources

**Decision**: The backend adds two aggregate reads (contract in
[contracts/openapi.yaml](./contracts/openapi.yaml)):

- `GET /api/v1/monthly-totals?year=2026`: twelve entries `{ month: "2026-10", total: Money,
  spendingCount }`, plus `year` and `today` (today's date in the app time zone, R-008). `year`
  defaults to the current year.
- `GET /api/v1/category-totals?from=2026-10-01&to=2026-10-31`: one entry per category with at
  least one spending in the range, `{ category: Ref, total: Money, spendingCount }`, largest
  total first (ties by name), plus the range `total` and `spendingCount`.

Both are single `GROUP BY` queries over `ix_spendings_user_date`. Both are unpaged and bounded
(12 rows; at most the user's category count), recorded under Complexity Tracking.

The frontend computes only presentation from these: whole-number share percentages and pie
angles, using integer cents parsed from the decimal strings (`utils.ts`), never floats of money.

**Rationale**: Feature 002 left totals out. Summing pages client-side would mean fetching a whole
year of spendings for the month line on every page view, breaking SC-005 at 10k spendings and
putting arithmetic on money in the frontend (constitution: backend owns rules; no float money).
A `GROUP BY` returns in milliseconds.

**Alternatives considered**:
- *Sum in the frontend from the spending list*: up to 100 items per page, so a busy year needs
  dozens of calls per render. Rejected.
- *One combined `/dashboard` resource*: couples unrelated views and is an action-shaped resource.
  Rejected.
- *Nested `/spendings/totals`*: `/spendings/{id}` already exists; `totals` would collide with
  the id segment. Rejected.

---

## R-003 — The month table loads every spending of the month

**Decision**: `getMonthSpendings(month)` in `data.ts` calls `GET /api/v1/spendings` with
`from`/`to` set to the month's bounds, `size=100`, `sort=date,desc`, and follows `totalPages`
until every page is read (normally one call). The table, its row count and its order come from
that list; the footer total and count come from `monthly-totals` so they match the month line.

**Rationale**: The spec shows all of a month's spendings in one table (Edge Cases). The existing
sort already breaks ties by creation time, which gives FR-020's "most recently recorded first
among the same date".

**Alternatives considered**: Paging the table in the UI (not in the design); raising the API's
page cap (a contract change for an edge case).

---

## R-004 — Saving with a typed category or account name: the backend resolves it

**Decision**: `SpendingRequest` gains optional `categoryName` and `accountName`. `categoryId`
and `accountId` become optional. Exactly one of each pair must be given (422 with a JSON-pointer
error otherwise). On create and update the service, in the same transaction:

1. trims the name and looks it up case-insensitively among the caller's items;
2. uses the existing item, or creates it (the same name rules and duplicate guard as
   `POST /accounts` / `POST /categories`);
3. saves the spending.

The frontend always sends names for a typed value and ids for a picked one. The idempotency key
covers the whole operation, so a retry neither duplicates the spending nor creates a second
category.

This replaces the spec's edge case "Partly created items on a failed save": nothing is created
when the spending is refused. The spec is updated to match.

**Rationale**: "Use the existing one if the name matches, otherwise create it" is a business rule
and belongs in the backend. Doing it in a Server Action would be a multi-call orchestration (the
BFF the constitution forbids), with a window where the category exists but the spending does
not, and a race when two tabs create the same name.

**Compatibility**: Turning required request fields into optional ones is allowed within v1
(Principle VII lists only the reverse as breaking). Responses are unchanged.

**Alternatives considered**:
- *Server Action calls `POST /categories` then `POST /spendings`*: partial writes, duplicate
  races, rule in the frontend. Rejected.
- *A new `POST /spendings-with-names`*: a verb-like resource duplicating `POST /spendings`.
  Rejected.

---

## R-005 — Merchant suggestions: a derived `merchants` resource

**Decision**: `GET /api/v1/merchants?limit=500` returns the caller's distinct merchants, grouped
by `lower(trim(merchant))`, each with:

- `name`: the spelling on its most recent spending;
- `spendingCount`;
- `lastUsedOn`;
- `lastCategory` and `lastAccount` (`Ref`): those of its most recent spending (FR-058).

Items are ordered by `spendingCount` desc, then `name`. `limit` defaults to 500, maximum 1000.
The response says whether it was truncated. One query (`DISTINCT ON` over a window), read-only.

The Spendings page loads it once per render and hands it to the entry row, which filters and
ranks locally as the user types.

**Rationale**: Autofill has to react on every keystroke. A server round trip per key would lag;
a local list does not. A user's distinct merchants number in the hundreds; 500 of the most used
covers autofill (rarely used merchants are simply typed in full).

**Alternatives considered**:
- *Search endpoint `?q=` per keystroke*: latency on every key, more load. Rejected.
- *Derive merchants from the loaded month only*: misses everything outside the month. Rejected.
- *Store merchants as an entity*: a migration and new write paths for derived data. Rejected
  (Principle VI).

---

## R-006 — Last-account rule on the backend

**Decision**: `AccountService.delete` adds a check after the in-use check: if the account is the
caller's only one, throw `UnprocessableEntityException` with
`details = { reason: "last-account" }` (422). The account row lock already taken in `delete`
plus a `count(*)` of the caller's accounts in the same transaction make it race-safe enough at
personal scale; two concurrent deletes of the last two accounts are serialised by locking all
the caller's account rows (`SELECT … FOR UPDATE` on `accounts WHERE user_id = ?`).

Order of refusals: not found (404) → in use (409 `in-use`) → last account (422 `last-account`).

**Rationale**: Clarified in the spec (FR-086). 422 because the request is understood and the
state cannot be changed by the client in a way that makes it succeed except by adding another
account; it is a business rule, not a conflict on the item itself.

**Compatibility**: A new refusal reason on an existing operation, documented in the contract.
Backward compatible.

**Alternatives considered**: 409 `conflict` (used for in-use and duplicates; would blur the
`reason` meanings for the client). Rejected.

---

## R-007 — Problem details reach the UI

**Decision**: `ApiError` (frontend `definitions.ts`) gains `reason?: string`,
`details?: Record<string, unknown>` and `errors?: { field: string; message: string }[]`, read in
`apiFetch` from the problem body. `app/lib/messages.ts` maps them to the spec's wording:

| From the API | Message |
|---|---|
| 409 `reason: duplicate-name` | "“{name}” already exists." |
| 409 `reason: in-use` (category) | "{name} is used by {n} spendings, so it can't be removed." |
| 409 `reason: in-use` (account) | "{name} has {n} spendings, so it can't be removed." |
| 422 `reason: last-account` | "You need at least one account." |
| 422 `errors[]` on `/date` | "Pick a date that is today or earlier." |
| 404 | "This item no longer exists." then the page refreshes |
| `ApiUnavailableError`, 5xx | "Couldn't reach the server. Your changes weren't saved." |

The row and Settings also check emptiness, positive amount and case-insensitive duplicates
locally, for instant feedback (the same wording). The server stays authoritative.

**Rationale**: The design's messages are specific; the backend already puts a machine-readable
`reason` in `details`. Local checks make keyboard entry feel instant.

---

## R-008 — "Today", currency and money arithmetic

**Decision**:

- "Today" comes from `monthly-totals.today` (R-002), computed by the backend in its configured
  zone. The page passes it to the calendar (future days disabled), the month line (future months
  greyed) and the default month. The frontend gets no time-zone setting of its own.
- The currency comes from the `Money` objects the API returns. New amounts are sent with the
  currency of `monthly-totals` totals (always present, even for zero totals).
- `utils.ts` parses decimal strings into integer cents (`"84.37"` → `8437`), sums and divides
  only cents, and formats with `Intl.NumberFormat(locale, { style: 'currency', currency })`. Month
  line totals round to whole units (FR-090).

**Rationale**: One source of truth for zone and currency (the backend). Integer cents fit safely
in JS numbers up to ~90 trillion units, far beyond the 99,999,999.99 cap.

**Alternatives considered**: An `APP_TIME_ZONE` env var on the frontend (two settings that can
drift); `Number.parseFloat` on amounts (float money, forbidden).

---

## R-009 — Interactive pieces are small client components; writes are Server Actions

**Decision**:

- Server Components render the page shell, month line, cards and the static table rows' data.
  One client component, `spendings-table.tsx`, owns the table's interaction state: which row is
  open, hover/focus actions, delete. Inside it, `entry-row.tsx`, `combobox.tsx`,
  `date-picker.tsx` are client leaves. The By category toggle and Settings dialog are separate
  client leaves.
- Writes are Server Actions in `app/lib/actions.ts`: `saveSpending`, `deleteSpending`,
  `createCategory`, `deleteCategory`, `createAccount`, `deleteAccount`. Each re-checks the
  session, validates with Zod, calls the API through `authedFetch`, then calls `refresh()` from
  `next/cache` so the dynamic page re-renders with fresh totals (FR-038, SC-003). `saveSpending`
  returns the saved spending's month; the row then `router.push`es `?month=` when it differs.
- The entry row is a real `<form action={…}>` with named inputs, so Enter submits natively and
  the action receives `FormData`. Keyboard handling (Tab/Enter rules, FR-053–FR-056) is layered
  on top in the client component.
- A save is guarded twice against duplicates: the submit is disabled while pending
  (`useActionState` pending flag), and an `Idempotency-Key` (a UUID created when the row opens,
  sent as a hidden field) goes to `POST /spendings` (FR-039). Updates use `PUT`, idempotent by
  nature.

**Rationale**: Follows the `frontend-developer` skill: server-first, client leaves only where
there are state and handlers, Server Actions for writes, no token in the browser. Per-user data
is never cached, so `refresh()` (not `updateTag`) is the right invalidation.

**Deviation from the skill**: the skill says forms should work without JavaScript. The entry
row submits without JavaScript only in a basic way (typed names, a text date). Autofill,
calendar, Tab/Enter rules and hover actions require JavaScript by nature. Recorded in the plan.

**Alternatives considered**: A client-side data store with `fetch` from the browser (would need
a token or a proxy route: forbidden); a full client page (loses streaming and server rendering).

---

## R-010 — Combobox, calendar and pie are built in-house; no new runtime dependency

**Decision**:

- **Combobox** (`combobox.tsx`): WAI-ARIA 1.2 combobox with `aria-autocomplete="both"`. Typing
  sets the input to the best match with the untyped remainder selected
  (`setSelectionRange`). Arrow keys move `aria-activedescendant`. Tab or Enter accepts.
  Backspace removes the remainder. Escape closes the list. The ranking function
  (`rankSuggestions` in `utils.ts`) puts prefix matches before contains-matches, each by count
  desc then name; it is pure and unit-tested.
- **Date picker** (`date-picker.tsx`): a button that opens a non-modal popover with a month grid
  (ARIA grid pattern), previous/next month, "Today", days after `today` disabled. The visible
  field also accepts typed `YYYY-MM-DD` or `DD/MM/YYYY`.
- **Pie** (`category-pie.tsx`): inline SVG donut, one `<path>` per category with spending
  (FR-072, no cap), arcs from integer cents. A 12-colour categorical palette assigned in rank
  order so neighbouring slices differ (and differ in lightness, for colour-blind users); past 12
  the palette repeats but never on adjacent slices. Each slice has a `<title>` and is focusable
  with the name, amount and share; the legend lists the same in order.

**Rationale**: Three small, well-defined widgets. A component or chart library would add more
code than it saves and fight the design's styling. Principle VI.

**Alternatives considered**: Headless UI / Downshift / react-day-picker / Recharts. Each is
viable, but each is a new dependency for one widget. Native `<input type="date">` does not match
the design and cannot show "Today" consistently across browsers.

---


**Revision (2026-10-28, post-implementation review)**: the calendar and the suggestion lists are
rendered in a React portal on `document.body` with `position: fixed`, placed under their field
(or above it when there is no room) by a small shared hook, `app/ui/spendings/use-floating.ts`.
The table card's `overflow` had clipped them when they were positioned inside it. Still no new
runtime dependency; CSS anchor positioning was not used because browser support is not yet
universal.

## R-011 — Settings is a native `<dialog>`

**Decision**: `settings-dialog.tsx` uses `<dialog>` with `showModal()`: built-in focus trapping,
Escape to close, inert background; a click on the backdrop closes it (FR-080). Sections on the
left are a vertical ARIA tablist (Categories, Accounts). Lists come from the page's already
loaded `categories` and `accounts`; adds and removes call the Server Actions, and `refresh()`
updates both the dialog and the entry row's options (FR-085).

**Rationale**: Native semantics cover most of the accessibility work.

---

## R-012 — Visual design tokens

**Decision**: Tailwind 4 `@theme` tokens in `globals.css` from the design canvas: ground
`#F3F4EF`, surface `#FFFFFF`, ink `#17191A`, muted `#5C625E` (4.5:1 on white), lines `#DDE0D8`
and `#ECEEE8`, brand `#1E5B45` (hover `#123B2C`), brand tint `#E4EEE8`, error `#A33A16`.

Fonts are loaded with `next/font/google` (self-hosted, so the CSP `font-src 'self'` still
holds): Bricolage Grotesque (headings), IBM Plex Sans (body), IBM Plex Mono (amounts and dates,
tabular figures). Inter (001) is retired.

**Rationale**: FR-003 asks for the design's look; tokens keep it in one place.

---

## R-013 — Tests

**Decision**:

**Frontend** (`pnpm typecheck && pnpm lint && pnpm test`, then `pnpm e2e`):

- Unit (Vitest): `utils.ts` (cents parsing and summing, share rounding, month bounds, month param
  parsing and clamping, `rankSuggestions`, pie arcs) and `messages.ts`.
- Data (Vitest + MSW): every new `data.ts` function, including multi-page month loading and
  error mapping in `apiFetch`.
- Actions (Vitest): Zod branches, API error → message mapping, `refresh()` called on success.
- Components (RTL + user-event): combobox (autofill, Backspace, arrows, Tab/Enter), entry row
  (the FR-053–FR-058 keyboard rules, merchant fill-in, validation messages, one save per action),
  date picker (future days disabled, Today), table (actions on hover and focus, delete), month
  line (future months disabled), by-category toggle and pie (no cap, no "Everything else"),
  settings dialog (messages for empty, duplicate, in-use, last account).
- E2E (Playwright, against a real backend and the Auth0 test user): one spec per user story, plus
  an axe scan of `/` and `/spendings` with the row, calendar and Settings open (SC-008). Adds the
  dev dependency `@axe-core/playwright`.

**Backend** (`./gradlew check`, 80/75 overall, 75 per package):

- Unit: `TotalsService`, `MerchantService`, `SpendingService` name resolution, `AccountService`
  last-account rule.
- ITs (Testcontainers Postgres): `MonthlyTotalsIT`, `CategoryTotalsIT`, `MerchantsIT`
  (grouping, latest spelling, latest refs, limit), `SpendingByNameIT` (create and update by name,
  case-insensitive reuse, one-of rule, idempotent retry creates one category), `LastAccountIT`,
  ownership cases added to `OwnershipIT`, and a performance IT for the three reads at 10k
  spendings (< 200 ms each against the 1 s SC-005 budget).

**Rationale**: Constitution IV; SC-006 requires e2e for every flow.

---

## R-014 — Branches and delivery

**Decision**: New `003-spending-screens` branches from `origin/main` in both service worktrees
(and in `hl-budget-spec`). Backend lands first (contract additions, last-account rule), then the
frontend copies `contracts/api.yaml`, regenerates types and builds the screens. No new
environment variables, secrets or chart changes. Deploys through the usual release → hl-helm PR
→ Argo CD path.

**Rationale**: Constitution workflow ("backend contract changes land first").
