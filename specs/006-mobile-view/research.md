# Research: Mobile View

**Feature**: [spec.md](./spec.md) | **Plan**: [plan.md](./plan.md) | **Date**: 2026-10-31

All decisions are frontend-only. The ground rule from the spec (FR-002) shapes every one of
them: **no existing desktop component, wording or behaviour changes**. The phone layout is
built from new components under `app/ui/phone/`. They reuse the pure helpers in `app/lib/`
(validation, CSV reading, money and date formatting) and the existing Server Actions, and do
not import or edit desktop components.

---

## R-001: How the app chooses the phone or desktop layout

**Decision**: The phone layout is chosen by CSS and confirmed on the client. The server always
renders the desktop tree, exactly as today, inside a `DesktopOnly` wrapper whose element is
`display: contents` at 640px and wider and `display: none` below 640px. Next to it, the server
renders a `PhoneOnly` wrapper that shows a phone skeleton below 640px and nothing at 640px and
wider. After hydration, a `useIsPhone()` hook (`useSyncExternalStore` over
`matchMedia('(width < 40rem)')`, server snapshot `null` = unknown) decides:

- **Desktop** (`false`): `PhoneOnly` renders nothing. The desktop tree is untouched; its only
  addition is one wrapper element with `display: contents`, which does not affect layout.
- **Phone** (`true`): `DesktopOnly` unmounts its children, so desktop client components such as
  the page-level Enter shortcut stop running. `PhoneOnly` mounts the phone app with the month's
  data.
- **Width crosses 640px** (rotation, window resize): the hook fires and the trees swap.

**Rationale**:
- The desktop's server-rendered HTML, streaming order (header outside `<Suspense>`) and
  behaviour stay as they are. This is the strongest guarantee of FR-002 short of a separate app.
- No real content is rendered twice, so desktop Playwright locators never see duplicate text,
  labels or ids. The pre-hydration phone skeleton is `display: none` at desktop widths and
  holds no text that desktop tests look for.
- It needs no user-agent sniffing, so a narrow desktop window and a phone behave the same.

**Alternatives considered**:
- *User-agent or `Sec-CH-UA-Mobile` hint on the server*: avoids the skeleton flash on phones.
  But reading `headers()` must happen inside `<Suspense>` (`cacheComponents: true`), and the
  desktop header sits outside it today. Moving it would change desktop streaming. A UA also
  cannot tell portrait from landscape.
- *Render both full trees and hide one with CSS*: no flash anywhere. But it duplicates every
  label and text in the DOM before hydration, which risks strict-mode locator failures in the
  existing desktop e2e suite (SC-005), plus duplicate form controls.
- *A separate `/m` route*: breaks "same address" (FR-001) and needs redirects on resize.
- *Responsive classes on the existing desktop components*: the layouts differ too much (table
  vs list, inline row vs full screen), and every edit would touch desktop code.

## R-002: The breakpoint

**Decision**: 640px (`40rem`, Tailwind's `sm`). The phone layout applies at `width < 40rem`.
CSS uses `max-sm:` / `sm:` variants and JavaScript uses `matchMedia('(width < 40rem)')`, so
both agree at exactly 639.98px vs 640px.

**Rationale**: This is the clarified answer (phones only). The desktop already reflows below
`xl` (FR-024 of 003), so 640px and wider still gets a working desktop layout.

## R-003: Month and year navigation

**Decision**: The Month and Year `<select>` elements navigate with
`router.push('/spendings?month=YYYY-MM', { scroll: false })` inside `startTransition`. This is
the same URL the desktop month line links to, so the server loads the month exactly as for
the desktop (`loadSpendingsPage`). Choosing the current year with a later month selected
clamps the month to the current month before navigating (FR-014). Future months are
`<option disabled>`.

The phone's view choices (Spendings / By category, Table / Pie), its open screen and its notice
live in a `PhoneUiProvider` placed **outside** the page's `<Suspense>`. This follows the
pattern `ImportProvider` set in 004 (R-010 there), so they survive month changes (FR-030).

**Rationale**: One loader, one URL scheme, and no new data path. Native `<select>` brings up
the phone's own wheel or picker (FR-011) and is 16px text (FR-005).

## R-004: Years in the Year dropdown

**Decision**: A new read helper, `getEarliestSpendingYear()`, calls the existing
`GET /api/v1/spendings?sort=date,asc&size=1&page=0` and takes the year of the first item. It
falls back to the current year when there are none. The year list runs from the current year
down to that year (FR-013), and always includes the year being shown.

The phone's server component calls it alongside a React-`cache`d `loadSpendingsPage`. The
desktop and phone server parts therefore share one set of month reads per request.

**Rationale**: No backend change. The endpoint already supports `sort=date,asc` (OpenAPI enum).

**Trade-off**: The server cannot know the layout, so desktop page loads also make this one
extra, small read. Nothing visible changes on the desktop. The alternative is a client-side
fetch after the phone mounts, which would make the year options flicker; this was rejected.

## R-005: Full-screen screens and bottom sheets

**Decision**: Each full-screen screen (Edit / Add / Fix row, Settings, Import CSV, Review
import) and each bottom sheet (account menu, delete confirmation) is a native `<dialog>` opened
with `showModal()`, the same technique as the desktop Settings window (003 R-011). This gives:
- an inert, unscrollable background (FR-101), a trapped focus and dialog semantics for screen
  readers;
- an `aria-label` or `aria-labelledby` for each dialog.

Screens are styled `inset-0 h-dvh w-full max-w-none`, and sheets are anchored to the bottom.
Tapping a sheet's backdrop closes it. Page scroll is locked with a rule scoped to phone dialogs
only: `html:has(dialog[data-phone][open]) { overflow: hidden }`. Desktop dialogs are therefore
unaffected.

One state machine in `PhoneUiProvider` holds which screen and sheet are open:
`list | edit | settings | import | review | fix-row`, plus `menu | confirm-delete | none`.

**Rationale**: Native modal dialogs solve focus, inertness and Escape without new
dependencies, and the project already relies on them.

## R-006: Phone back gesture and browser Back

**Decision**: Opening a screen or sheet pushes one history entry at the current URL with
`window.history.pushState({ hlbPhone: depth }, '')`. Next 16 integrates native `pushState` with
its router and keeps the router's state on the entry. A `popstate` listener closes the topmost
screen or sheet when the depth drops. Closing by X, Cancel, backdrop or after a save calls
`history.back()` to consume the entry; a navigation that follows the close (switching to the
saved spending's month) runs only after the matching `popstate`. Overlays are never in the URL,
so a reload shows the Spendings list (clarified; FR-103).

When the layout switches to desktop while overlays are open, the phone tree pops its remaining
entries (`history.go(-depth)`) as it unmounts.

**Rationale**: Native history entries are the only way to make the Android back gesture and
iOS swipe-back close a screen instead of leaving the app.

**Risk**: This relies on how the App Router handles `popstate` back to entries it did not create
itself. A Playwright phone spec (`page.goBack()` on each screen) pins the behaviour, and a
unit test covers the depth bookkeeping.

## R-007: Category, account and merchant suggestions

**Decision**: A new phone `SuggestInput`: a text input (16px) with an inline list of up to 6
tappable options (each 44px tall) below it while it has focus and text. The options come from
the existing `rankSuggestions` helper (most used first, ties alphabetical) over the same lookup
data as the desktop. When empty and focused, it shows the 6 most used.

- Picking a merchant fills an empty category and account from the merchant's `lastCategory` and
  `lastAccount` (FR-046, clarified).
- A typed name that matches nothing is kept and created on save by the existing
  `saveSpending` action (FR-048).

**Rationale**: The design's `<datalist>` cannot rank by use, cannot tell when a value was
picked (needed for the merchant fill), and renders inconsistently on iOS. The desktop
`Combobox` is built around inline keyboard completion (selected remainder, Tab/Enter), which
the phone does not need, and editing it would risk desktop behaviour.

## R-008: Amount field

**Decision**: `<input type="text" inputMode="decimal" autoComplete="off">` at 28px or larger,
parsed with the existing `parseAmountInput`. Not `type="number"`: it accepts `e`, varies by
locale, and loses the typed text on invalid input.

## R-009: Calendar

**Decision**: A new inline `PhoneCalendar` (in the flow of the form, not a popover), built on
the existing `calendarWeeks`, `addMonths` and `isValidDate` helpers. Each day is a 44px
button, days after today are `disabled`, and picking a day sets the date and closes it. It has
previous and next arrows plus Today and Close (FR-044).

**Rationale**: The desktop `DatePicker` is a floating popover with keyboard grid navigation;
reusing it would mean changing a desktop component. The shared logic is already in `app/lib`.

## R-010: Validation order and messages

**Decision**: The phone uses `draftProblems()` from `app/lib/draft.ts` (the same rules and
messages as the desktop) and shows the first problem in the phone's field order:
`amount, date, merchant, category, account` (FR-047). The order is a phone-side constant; the
desktop `FIELDS` order is untouched. The message shows inline under the form as a
`role="alert"` paragraph, and focus moves to that field. A future date reads
`MESSAGES.dateNotFuture`, as on the desktop.

## R-011: Saving, deleting and importing

**Decision**: The phone calls the existing Server Actions unchanged:
- `saveSpending` (Idempotency-Key on add; returns the saved month);
- `deleteSpending`;
- `createCategory`, `deleteCategory`, `createAccount`, `deleteAccount`;
- `importSpendings` (all or nothing, Idempotency-Key, returns the latest month).

Buttons are disabled while an action is pending, so a save happens once per tap (FR-050,
FR-096). Error messages come from the action results, which already use the desktop's wording.

**Rationale**: The business rules stay in one place (Constitution II). There are no new
endpoints or actions.

## R-012: CSV reading on the phone

**Decision**: The phone uses `checkFile` and `readImport` / `rowsFromCsv` from
`app/lib/csv-import.ts`, so it applies the same header matching, quoting, 1 MB / 1,000-row
limits, day-first dates (clarified) and amount normalisation, with the same messages. A
negative amount is read as positive by the existing `parseImportAmount`; the test fixture
`"-18.90"` pins it. The file input uses `accept=".csv,text/csv"` and there is no drop zone.

The review's row status (warning or check icon, the amber "No date / Merchant missing / No
category / No account / No amount" texts) is derived per row from `draftProblems()`. A date
after today is shown in amber with its date text.

## R-013: Pie chart: top 6 plus "Everything else"

**Decision**: A new pure helper, `topCategories(items, 6)`, in `app/lib/phone.ts`. It keeps the
6 largest categories, sums the rest into "Everything else" when there are more than 6, and
returns share percentages from the existing `sharePercent`. Slices are drawn with the existing
`donutArcs`. They use the first 6 colours from `CATEGORY_COLORS`, the same palette as the
desktop, and a neutral `--color-muted` for "Everything else". Table view lists every category
(FR-031).

## R-014: Touch, iOS and the keyboard

**Decision**:
- **Touch targets**: interactive elements get `min-h-11 min-w-11` (44px), and list rows are
  full-width buttons at least 56px tall (FR-004).
- **Text size**: every `input`, `select` and `textarea` in the phone tree uses `text-base`
  (16px) or larger (FR-005).
- **No hover**: no hover-only affordances. `hover:` styles are wrapped in
  `@media (hover: hover)` by Tailwind v4 by default.
- **Safe areas**: the root layout's `viewport` export adds `viewportFit: 'cover'`, and the
  floating button and bottom bars pad with `env(safe-area-inset-bottom)`. Desktop browsers
  ignore both.
- **Keyboard overlap**: the Edit screen is a flex column (`h-dvh`) with a scrolling body and the
  action bar at the end. `interactiveWidget: 'resizes-content'` makes Android Chrome shrink the
  page for the keyboard. On iOS, a small hook reads `visualViewport` and sets
  `--keyboard-inset`, so the bar sits above the keyboard (FR-040).
- **Floating button**: `position: fixed` bottom-right, 56px tall. The list ends with a spacer
  of the same height plus the safe area, so the last row is never covered (FR-020).

## R-015: Signed-out screen on the phone

**Decision**: The existing `/` signed-out screen is a single centred card. Only `max-sm:`
variants are added to it, where needed for the 44px Sign in button and phone padding, so its
desktop classes stay the same. Signing out from the phone menu uses the same plain
`/auth/logout` link as the desktop header.

## R-016: Testing strategy

**Decision**:
- **Unit (Vitest + Testing Library)**:
  - every new phone component;
  - `topCategories`, the year range, the phone validation order and the history-depth logic;
  - `useIsPhone` with a mocked `matchMedia`.
  Existing tests are not edited.
- **E2E (Playwright)**:
  - A second project, `phone`: `devices['iPhone 13']` with `browserName: 'chromium'` (390px,
    touch, mobile UA), so CI needs no WebKit. Phone specs live in `e2e/phone/`. The existing
    `chromium` project gets `testIgnore: /phone\//`, so desktop specs run exactly as before.
  - Phone specs cover every user story: month view, add, edit, delete with the sheet, by
    category, the menu with settings and sign-out, import with fixing rows, and the back
    gesture.
  - Shared checks run on every phone screen and sheet:
    - every visible control's bounding box is at least 44×44 (SC-003);
    - computed `font-size` is at least 16px on inputs and selects (SC-003);
    - `scrollWidth <= clientWidth` at 320, 390 and 430px (SC-002);
    - an axe scan has no serious or critical issues (SC-006).
- **Desktop guard (SC-005)**: the full existing unit and desktop e2e suites must pass with
  **no edits to existing spec files**. Quickstart §4 adds a by-hand before/after screenshot
  comparison at 1280px and 1440px.
