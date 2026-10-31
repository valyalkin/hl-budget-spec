---

description: "Task list for 006-mobile-view"
---

# Tasks: Mobile View

**Input**: Design documents from `/specs/006-mobile-view/`

**Prerequisites**: plan.md, spec.md, research.md, data-model.md, contracts/phone-ui.md,
quickstart.md

**Tests**: REQUIRED. Constitution Principle IV requires that tests ship with the code and that
every new user-visible flow gets a Playwright spec. Research R-016 lists them. Within each
phase, write the tests first and see them fail before writing the code that makes them pass.

**Organization**: Tasks are grouped by user story (spec.md US1–US6), so each story can be built
and checked on its own. Only the frontend changes; there is no backend work.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Can run in parallel (different files, no dependency on an unfinished task).
- **[Story]**: The user story the task belongs to (US1 … US6).
- Every task ends with `· skill: <name>`. Load that skill before starting the task; its
  definition of done is part of the task's done. Where a skill and the constitution disagree,
  the constitution wins.

## Path Conventions

All paths are relative to the spec repo root `/Users/valyalkin/Documents/Projects/hl-budget-spec/`.

- `hl-budget-frontend/` is the git worktree of `valyalkin/hl-budget-frontend`, switched to
  branch `006-mobile-view` (T001). It is written below as `fe/`.
- **Commits**: commit inside `fe/` using Conventional Commits, after each task or logical
  group.
- **Merging**: merging to `main` triggers a release. Do it only after the gates are green and
  the user agrees.

Shared rules that several tasks reference:

- **DESKTOP MUST NOT CHANGE (FR-002)**. This is the rule every task is checked against:
  - Do not edit any file under `fe/app/ui/spendings/`, `fe/app/ui/import/`,
    `fe/app/ui/settings/`, or `fe/app/ui/app-header.tsx`.
  - Do not edit any existing `fe/e2e/*.spec.ts` or existing `*.test.ts(x)` file.
  - Phone code may import from `fe/app/lib/` and from `fe/app/ui/spendings/category-colors.ts`
    (constants only), but never desktop components.
  - The only existing files that may change are those listed in plan.md "Structure Decision".
- **Phone code location**: components in `fe/app/ui/phone/`, pure helpers in
  `fe/app/lib/phone.ts`, and phone e2e specs in `fe/e2e/phone/`.
- **Breakpoint**: the phone layout applies at `width < 40rem` (640px). In CSS, use `max-sm:` or
  `sm:`; in JavaScript, use `matchMedia('(width < 40rem)')` (R-002).
- **Touch and text (FR-004 – FR-006)**:
  - Every phone control is at least 44×44 CSS px (`min-h-11 min-w-11`).
  - Every phone `input` or `select` uses `text-base` (16px) or larger.
  - No hover-only affordance.
- **Money**: amounts are decimal strings. Convert to cents only with `toCents`, and back with
  `formatCents` or `formatMoney` from `fe/app/lib/utils.ts`. Never use `parseFloat` or
  `Number(…)` on money.
- **Messages**: every new user-facing string goes in `fe/app/lib/messages.ts` (additions only),
  worded exactly as in spec.md and contracts/phone-ui.md. Reuse the existing desktop messages
  for validation, Settings and import errors.
- **Server Actions**: call the existing actions in `fe/app/lib/actions.ts` unchanged. A
  `signed-out` result calls `window.location.assign('/')`. Disable the triggering button while
  an action is pending, so there is one call per tap (FR-050, FR-096).
- **Client components**: put `'use client'` only on phone components that need state or
  effects. No `useMemo`, `useCallback` or `React.memo` (the React Compiler is on).
- **Dialogs**: every phone screen and sheet is a `PhoneDialog` (T012), never a hand-made
  overlay `div`.

---

## Phase 1: Setup

**Purpose**: Get a fresh feature branch with green gates, and a desktop baseline to compare
against later.

- [X] T001 In `hl-budget-frontend/`:
  1. Run `git fetch origin` and `git switch -c 006-mobile-view origin/main`. The worktree is
     currently on the merged `fix/e2e-import-live` branch.
  2. Run `pnpm install && pnpm typecheck && pnpm lint && pnpm test` and confirm they are green.
  3. Run `pnpm e2e --project=chromium` and confirm it is green. Record the pass count in this
     file's Implementation notes as the desktop baseline.

  The spec repo is already on `006-mobile-view`. Ask the user before committing
  `specs/006-mobile-view/` · skill: frontend-developer:frontend-developer
- [X] T002 Take the desktop baseline screenshots from quickstart.md §4: the Spendings screen,
  an open entry row, the Settings window, Import CSV and Review import, each at 1280px and
  1440px, from `origin/main` running locally. Save them under
  `specs/006-mobile-view/baseline/` (git-ignored; add the folder to the spec repo's
  `.gitignore`) · skill: none

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: Build the layout switch, the phone shell, shared dialogs and helpers, and the
phone test project. No story can start until this phase is done.

**Checkpoint**: At 390px the signed-in `/spendings` shows an empty phone shell (the
`data-layout="phone"` root). At 1280px the desktop is pixel-identical to T002, and
`pnpm e2e --project=chromium` passes unchanged.

### Tests first (see them fail)

- [X] T003 [P] Create `fe/app/ui/phone/use-is-phone.test.ts`. Mock `window.matchMedia` and
  check that `useIsPhone()`:
  - returns `null` on the server snapshot;
  - returns `true` for `(width < 40rem)` matching and `false` otherwise;
  - updates when the media query's `change` event fires;
  - removes its listener on unmount.

  · skill: frontend-developer:frontend-developer
- [X] T004 [P] Create `fe/app/ui/phone/layout-switch.test.tsx` covering:
  - `DesktopOnly` renders its children inside one element with classes
    `contents max-sm:hidden` while `useIsPhone()` is `null` or `false`, and renders nothing
    when it is `true`;
  - `PhoneOnly` renders the `PhoneSkeleton` (inside an `sm:hidden` element) while `null`,
    nothing when `false`, and its children when `true`.

  · skill: frontend-developer:frontend-developer
- [X] T005 [P] Create `fe/app/ui/phone/phone-ui-provider.test.tsx` for the state machine in
  data-model.md, "Phone UI state":
  - **Initial state**: `screen 'list'`, `sheet 'none'`, `view 'spendings'`, `chart 'table'`,
    empty `notice`, `historyDepth 0`.
  - **Opening**: `openScreen` / `openSheet` push exactly one history entry each
    (`history.pushState` spy) and increase `historyDepth`. `menu` opens only over `list`;
    `confirm-delete` opens only over `edit`.
  - **Closing**: `close()` calls `history.back()`, and the layer closes on the resulting
    `popstate`. A `popstate` with no `close()` call (the back gesture) closes the topmost layer.
    `review` + back gives `import` (rows discarded); `fix-row` + back gives `review`.
  - **`closeThen(fn)`**: runs `fn` only after the matching `popstate`.
  - **Notice**: set by `notify(text)`; cleared by the next `openScreen`, `openSheet`,
    `setView`, `setChart` or month change.
  - **Unmount** with `historyDepth > 0` calls `history.go(-historyDepth)`.

  · skill: frontend-developer:frontend-developer
- [X] T006 [P] Create `fe/app/ui/phone/phone-dialog.test.tsx`. `PhoneDialog` must:
  - render a native `<dialog data-phone>` opened with `showModal()`; jsdom needs a stub for
    `HTMLDialogElement.prototype.showModal` and `close`;
  - support `variant="screen"` (full screen) and `variant="sheet"` (bottom), and be labelled by
    `label` or `labelledBy`;
  - call `onDismiss` on Escape (`cancel` event; prevent the default close), and on a backdrop
    click for sheets only.

  · skill: frontend-developer:frontend-developer
- [X] T007 [P] Create `fe/app/lib/phone.test.ts` for the helpers of T010, starting with:
  - **`monthOptions(year, today)`**: 12 options with labels "January" … "December"; in
    today's year, months after today's are `disabled`.
  - **`yearOptions(today, earliestYear, shownYear)`**: newest first, from `today`'s year down to
    `min(earliestYear, shownYear)`, and at least the current year. Example:
    `('2026-10-31', 2025, 2026)` gives `[2026, 2025]`.
  - **`clampMonth({year, month}, today)`**: the current year with a later month gives today's
    month; other inputs are unchanged.

  Later stories extend this file · skill: frontend-developer:frontend-developer
- [X] T008 [P] Extend `fe/app/lib/data.test.ts` with new cases only; do not edit existing ones:
  - `getEarliestSpendingYear()` calls `/api/v1/spendings?sort=date,asc&size=1&page=0` (MSW)
    and returns the first item's year;
  - it returns `null` for an empty page;
  - `loadSpendingsPage` called twice with the same argument in one render makes one set of
    reads (React `cache`).

  · skill: frontend-developer:frontend-developer

### Implementation

- [X] T009 [P] Create `fe/app/ui/phone/use-is-phone.ts`: `useIsPhone(): boolean | null`, using
  `useSyncExternalStore` over `matchMedia('(width < 40rem)')`, with a server snapshot of `null`
  (R-001). Make T003 pass · skill: frontend-developer:frontend-developer
- [X] T010 [P] Create `fe/app/lib/phone.ts` with `monthOptions`, `yearOptions` and
  `clampMonth`, using `monthLabel` and `monthOf` from `utils.ts` where they fit. Make T007
  pass. Later stories add helpers here · skill: frontend-developer:frontend-developer
- [X] T011 [P] Create the layout wrappers and skeleton (makes T004 pass):
  - `fe/app/ui/phone/layout-switch.tsx` (`'use client'`), exporting `DesktopOnly` and
    `PhoneOnly` as described in T004;
  - `fe/app/ui/phone/phone-skeleton.tsx`: a header bar, two select-shaped blocks, a total
    block and 6 row blocks, all `aria-hidden`, with a visually hidden "Loading spendings" status
    and no other text.

  · skill: frontend-developer:frontend-developer
- [X] T012 [P] Create `fe/app/ui/phone/phone-dialog.tsx` (`'use client'`) per T006 and R-005:
  - **Screen variant**: `fixed inset-0 m-0 h-dvh max-h-none w-full max-w-none`, a flex column.
  - **Sheet variant**: anchored to the bottom, rounded top corners, a grab handle
    (`aria-hidden`), and `pb-[env(safe-area-inset-bottom)]`.
  - **Backdrop**: dimmed.

  Make T006 pass · skill: frontend-developer:frontend-developer
- [X] T013 Create `fe/app/ui/phone/phone-ui-provider.tsx` (`'use client'`) with the state,
  actions and history bookkeeping of T005 and R-006. Use
  `window.history.pushState({ hlbPhone: depth }, '')`, a single `popstate` listener, and
  `history.go(-depth)` on unmount. Export `usePhoneUi()`. Make T005 pass. Depends on T012 ·
  skill: frontend-developer:frontend-developer
- [X] T014 In `fe/app/lib/data.ts`, make two additive changes:
  - wrap `loadSpendingsPage` in React `cache` without changing its signature or behaviour;
  - add `getEarliestSpendingYear(): Promise<number | null>` per R-004, using the existing
    `read` helper.

  Make T008 pass, and make sure the existing `data.test.ts` cases still pass untouched ·
  skill: frontend-developer:frontend-developer
- [X] T015 Create the phone section and the app shell. Depends on T013 and T014.
  - `fe/app/ui/phone/phone-section.tsx` (server component): awaits `searchParams`, calls the
    cached `loadSpendingsPage(month)` and `getEarliestSpendingYear()` in parallel, and passes
    plain data to `PhoneApp`, mapped the same way `month-section.tsx` maps categories and
    merchants. Copy that mapping; don't import it.
  - `fe/app/ui/phone/phone-app.tsx` (`'use client'`): the root
    `<div data-layout="phone" className="min-h-dvh bg-ground">`. It is an empty shell for now.

  · skill: frontend-developer:frontend-developer
- [X] T016 Wire the switch into `fe/app/spendings/page.tsx`. Keep the existing tree exactly,
  wrapped:

  ```tsx
  <ImportProvider><PhoneUiProvider>
    <DesktopOnly>{existing AppHeader, ImportNotice, Suspense+MonthSection}</DesktopOnly>
    <PhoneOnly><Suspense fallback={<PhoneSkeleton/>}><PhoneSection searchParams={props.searchParams}/></Suspense></PhoneOnly>
  </PhoneUiProvider></ImportProvider>
  ```

  Then check the result at 1280px against T002 (identical) and run
  `pnpm e2e --project=chromium` (green, same count as T001) · skill:
  frontend-developer:frontend-developer
- [X] T017 [P] Add the phone-wide settings (R-014):
  - In `fe/app/layout.tsx`, add
    `export const viewport: Viewport = { width: 'device-width', initialScale: 1, viewportFit: 'cover', interactiveWidget: 'resizes-content' }`.
  - In `fe/app/globals.css`, add only:
    - `html:has(dialog[data-phone][open]) { overflow: hidden; }`
    - a `.pb-safe` utility for `env(safe-area-inset-bottom)`.

  Confirm the desktop at 1280px still matches T002 · skill:
  frontend-developer:frontend-developer
- [X] T018 [P] Create `fe/app/ui/phone/use-keyboard-inset.ts`: while mounted, it sets
  `--keyboard-inset` on `document.documentElement` from
  `window.innerHeight - visualViewport.height - visualViewport.offsetTop` (min 0), updated on
  `visualViewport` `resize` and `scroll`, and removes it on unmount. Include a unit test in
  `fe/app/ui/phone/use-keyboard-inset.test.ts` · skill: frontend-developer:frontend-developer
- [X] T019 Set up the phone e2e project:
  - **`fe/playwright.config.ts`**: add `testIgnore: /phone\//` to the existing `chromium`
    project, and change nothing else about it. Add a project
    `{ name: 'phone', testMatch: /phone\/.*\.spec\.ts/, use: { ...devices['iPhone 13'], browserName: 'chromium' } }`.
  - **`fe/e2e/phone/support.ts`**: re-export `signIn`, `marker` and `E2E_ACCOUNT` from
    `../support`. Add these helpers:
    - `expectPhoneLayout(page)`: `[data-layout="phone"]` is visible.
    - `assertTouchTargets(page)`: every visible `button, a, input, select, [role=option]` inside
      the open dialog, or else the phone root, has a bounding box of at least 44×44.
    - `assertNoZoomInputs(page)`: computed `font-size` is at least 16px on every visible input
      and select.
    - `assertNoSideScroll(page)`: `scrollWidth <= clientWidth`.
    - `checkScreen(page)`: runs all three checks above.

    Note: the shared `signIn` waits for the desktop month heading, so the phone sign-in helper
    must wait for `expectPhoneLayout` instead. Write it in `phone/support.ts`; don't edit
    `../support.ts`.

  · skill: frontend-developer:frontend-developer

---

## Phase 3: User Story 1 — See my spendings for a month on the phone (Priority: P1) 🎯 MVP

**Goal**: At 390px the user sees the header, the Month and Year dropdowns, the month total and
count, the Spendings | By category switch, and the month's list (date, category, amount).
Changing month or year reloads the list.

**Independent Test**: With spendings in September and October, open the app at 390px. October
is selected and the list, total and count match. Pick September and they update (spec US1).

### Tests first

- [X] T020 [P] [US1] Create `fe/e2e/phone/month-view.spec.ts` covering US1 scenarios 1–7.
  - **Current month**: it is selected; total, count and rows match the data (seed with a
    marker merchant).
  - **Future months**: they are disabled in "Month".
  - **Year clamp**: pick December of last year, then switch "Year" to the current year; the
    month becomes the current month.
  - **Empty month**: it shows "No spendings in [Month Year]."
  - **Floating button**: "Add spending" stays in the viewport after scrolling to the bottom,
    and does not overlap the last row's bounding box.
  - **Signed out**: a signed-out visitor at 390px sees "You're signed out" and "Sign in".
  - **Screen checks**: `checkScreen` at 320, 390 and 430px widths.

  · skill: frontend-developer:frontend-developer
- [X] T021 [P] [US1] Create unit tests for the US1 components:
  - `fe/app/ui/phone/phone-header.test.tsx`: "HL-Budget" text, and an "Account menu" button with
    `aria-haspopup="dialog"`.
  - `fe/app/ui/phone/month-year-picker.test.tsx`:
    - selects labelled "Month" and "Year", options from T010;
    - changing either calls `router.push('/spendings?month=YYYY-MM', { scroll: false })`, with
      `next/navigation` mocked, and applies `clampMonth`.
  - `fe/app/ui/phone/spending-list.test.tsx`:
    - rows are buttons named "Oct 27, Groceries, $64.38, Neighbourhood Market" (see notes) showing only the
      date, category and amount, newest first;
    - the empty text;
    - the "Add spending" button;
    - a long category is truncated (`truncate` class) while the amount stays visible.

  · skill: frontend-developer:frontend-developer

### Implementation

- [X] T022 [P] [US1] Create `fe/app/ui/phone/phone-header.tsx`: the logo (`aria-hidden`) and
  "HL-Budget" on the left; a 44px round "Account menu" button on the right, calling
  `openSheet('menu')`. The menu itself comes in US5; until then the button opens nothing
  visible · skill: frontend-developer:frontend-developer
- [X] T023 [P] [US1] Create `fe/app/ui/phone/month-year-picker.tsx` (`'use client'`): two
  native `<select>`s side by side (`grid-cols-2`, `text-base`, at least 44px), navigating per
  R-003 inside `startTransition`. Below them:
  - the month total in `font-mono text-4xl`, which must fit $100,000.00 on one line at 320px
    (use `text-[clamp(...)]` if needed);
  - the count via `plural(n, 'spending', 'spendings')`;
  - the "View" group with "Spendings" and "By category" (`aria-pressed`) bound to `setView`.

  · skill: frontend-developer:frontend-developer
- [X] T024 [P] [US1] Create `fe/app/ui/phone/spending-list.tsx` (`'use client'`):
  - **Rows**: full-width `<button>` rows at least 56px tall: `shortDate(date)` | category
    (truncate) | `formatMoney(amount)`, right-aligned and monospaced. `aria-label` per
    contracts/phone-ui.md. A tap calls `openScreen('edit', { spendingId })`; the screen comes
    in US3.
  - **Empty state**: "No spendings in [Month Year]." (message added to `messages.ts`).
  - **Floating button**: a fixed bottom-right "Add spending" pill, 56px, at
    `bottom-[calc(1rem+env(safe-area-inset-bottom))]`, calling `openScreen('edit', { mode: 'add' })`.
  - **End spacer**: matching height, so the last row is never covered.

  · skill: frontend-developer:frontend-developer
- [X] T025 [US1] Compose the main screen in `fe/app/ui/phone/phone-app.tsx`: the header, then
  the picker, then the `role="status"` notice (when set), then `SpendingList` when `view` is
  `'spendings'`. Make T020 (except its signed-out part) and T021 pass. Depends on T022–T024 ·
  skill: frontend-developer:frontend-developer
- [X] T026 [US1] Signed-out screen on phones (R-015): check `fe/app/ui/signed-out.tsx` at 320
  and 390px. Where the Sign in button is under 44px tall or the card overflows, add `max-sm:`
  classes only; do not change any existing class. Make the signed-out part of T020 pass, and
  confirm the desktop signed-out page is unchanged · skill: frontend-developer:frontend-developer

**Checkpoint**: US1 works at 390px; desktop e2e still green.

---

## Phase 4: User Story 2 — Add a spending on the phone (Priority: P1)

**Goal**: The floating "Add spending" button opens the full-screen New spending screen: amount
first, an inline calendar, type-or-pick merchant, category and account, validation, and saving.

**Independent Test**: At 390px, add 12.40 for today with a typed merchant and existing category
and account. It appears in the list and on the server with those exact values (spec US2).

### Tests first

- [X] T027 [P] [US2] Create `fe/e2e/phone/add-spending.spec.ts` covering US2 scenarios 1–11:
  - **Layout**: field order and labels; the amount input has `inputmode="decimal"`; the date
    defaults to today in the current month and to the 1st of a past month.
  - **Calendar**: it opens inline, future days are disabled, and "Today" and "Close" work.
  - **Merchant fill**: picking a past merchant fills empty Category and Account but keeps
    filled ones.
  - **New names**: a new category name typed in any case is created; a differently-cased
    existing one is reused.
  - **Validation**: each message appears in phone field order (empty amount first).
  - **Save**: it lands on the spending's month, and a double tap saves once.
  - **Close**: X saves nothing.
  - **Screen checks**: `checkScreen` on the form and on the open calendar.

  Clean up the marker spendings at the end · skill: frontend-developer:frontend-developer
- [X] T028 [P] [US2] Extend `fe/app/lib/phone.test.ts`:
  - `PHONE_FIELDS` equals `['amount','date','merchant','category','account']`.
  - `firstPhoneProblem(draft, today)` returns the first `draftProblems` entry in that order.
    Example: an empty draft gives `{ field: 'amount', message: 'Enter an amount greater than zero.' }`.
    A date after today gives the desktop's `dateNotFuture` message.

  · skill: frontend-developer:frontend-developer
- [X] T029 [P] [US2] Create unit tests for the US2 components:
  - `fe/app/ui/phone/phone-calendar.test.tsx`:
    - the month title; "Previous month" and "Next month";
    - day buttons named like "Tuesday, October 27, 2026", with the selected day
      `aria-pressed="true"` and days after `today` `disabled`;
    - "Today" picks today and closes; "Close" closes without changing the date;
    - every day button has `min-h-11`.
  - `fe/app/ui/phone/suggest-input.test.tsx`:
    - an empty focus shows the 6 most used;
    - typing narrows via `rankSuggestions` (most used first, ties alphabetical), at most 6
      options of role `option`;
    - a tap picks the option and calls `onPick(item)`; typed text with no match is kept;
    - the input has `text-base`.
  - `fe/app/ui/phone/spending-form.test.tsx` (add mode):
    - title "New spending" and button "Add spending";
    - field order;
    - the inline `role="alert"` message and focus on the problem field;
    - `saveSpending` is called once with `{ draft, idempotencyKey }` (mocked) and
      `closeThen` navigates to the returned month;
    - a server error keeps the form and shows the message;
    - the merchant pick fills empty fields only.

  · skill: frontend-developer:frontend-developer

### Implementation

- [X] T030 [P] [US2] In `fe/app/lib/phone.ts`, add `PHONE_FIELDS` and `firstPhoneProblem`
  (R-010), built on `draftProblems` from `draft.ts`. Make T028 pass · skill:
  frontend-developer:frontend-developer
- [X] T031 [P] [US2] Create `fe/app/ui/phone/phone-calendar.tsx` (`'use client'`), the inline
  calendar of R-009, using `calendarWeeks`, `addMonths`, `longDate` and the weekday labels
  "Su" … "Sa". Picking a day calls `onPick(iso)`, and the parent closes the calendar. Make the
  calendar part of T029 pass · skill: frontend-developer:frontend-developer
- [X] T032 [P] [US2] Create `fe/app/ui/phone/suggest-input.tsx` (`'use client'`), the
  type-or-pick combobox of R-007 (`role="combobox"`, `aria-expanded`, `aria-controls`; a
  listbox of 44px `option` buttons). Make the suggest part of T029 pass · skill:
  frontend-developer:frontend-developer
- [X] T033 [US2] Create `fe/app/ui/phone/spending-form.tsx` (`'use client'`) with
  `mode: 'add' | 'edit' | 'fix-row'`. This task implements `add`.
  - **Frame**: a `PhoneDialog variant="screen"` labelled by its title, with "Close without
    saving" (X) at the top left.
  - **Body** (scrolling): the Amount (CAD) field with a "$" prefix, `inputMode="decimal"` and
    `text-3xl`, then Date (a button showing `longDate`, toggling the inline calendar), then
    Merchant, Category and Account (`SuggestInput`). Category options are the defaults plus the
    user's own; merchant options carry `lastCategory` and `lastAccount`.
  - **Footer**: the bottom bar is the last child, with
    `pb-[calc(1rem+env(safe-area-inset-bottom))]` and
    `mb-[var(--keyboard-inset,0px)]`; use `useKeyboardInset` (T018).
  - **Default date**: `defaultDateFor(ym, today)`.
  - **Save**: `firstPhoneProblem`; on success, `saveSpending`, then
    `closeThen(() => router.push('/spendings?month=' + month))`.

  Make the add part of T029 pass · skill: frontend-developer:frontend-developer
- [X] T034 [US2] In `fe/app/ui/phone/phone-app.tsx`, render `SpendingForm` when
  `screen === 'edit'` in add mode, passing the lookups and `ym`/`today`. Make T027 pass.
  Depends on T033 · skill: frontend-developer:frontend-developer

**Checkpoint**: US1 and US2 together are a usable phone MVP: view and add.

---

## Phase 5: User Story 3 — Edit or delete a spending on the phone (Priority: P1)

**Goal**: Tapping a row opens "Edit spending" with a red-outlined "Delete spending" button.
Deleting asks in a bottom sheet, then returns with a "Deleted …" note.

**Independent Test**: At 390px, edit a spending's amount and save; then delete it through the
sheet. It is gone and "Deleted …" shows (spec US3).

### Tests first

- [X] T035 [P] [US3] Create `fe/e2e/phone/edit-delete.spec.ts` covering US3 scenarios 1–7:
  - **Edit**: the fields are prefilled; save shows the change; X keeps the old values.
  - **Delete sheet**: it shows "[merchant] · [amount] · [Mon D, YYYY]". Cancel and a backdrop
    tap keep the spending with Edit still open.
  - **Delete**: it removes the spending and shows "Deleted [merchant] · [amount]". The note
    clears after the next tap.
  - **Screen checks**: `checkScreen` on the sheet.

  · skill: frontend-developer:frontend-developer
- [X] T036 [P] [US3] Extend `fe/app/ui/phone/spending-form.test.tsx` (new cases) and create
  `fe/app/ui/phone/delete-sheet.test.tsx`.
  - **Edit mode**: title "Edit spending"; button "Save changes"; prefilled values;
    `saveSpending` gets the spending id; a red-outlined "Delete spending" button that
    calls `openSheet('confirm-delete')`. Add mode has no delete button.
  - **Delete sheet**:
    - an `alertdialog` labelled "Delete this spending?" with the body text;
    - "Delete spending" calls `deleteSpending` once, then `closeThen` back to the list and
      `notify('Deleted …')`;
    - a failure keeps the spending and shows the action's message;
    - "Cancel" and the backdrop ("Keep this spending") close only the sheet.

  · skill: frontend-developer:frontend-developer

### Implementation

- [X] T037 [US3] In `fe/app/ui/phone/spending-form.tsx`, implement `mode: 'edit'`:
  - prefill the fields from the spending;
  - "Save changes";
  - a red-outlined "Delete spending" button (at least 44px) below the fields;
  - after a save, go to the spending's month.

  · skill: frontend-developer:frontend-developer
- [X] T038 [P] [US3] Create `fe/app/ui/phone/delete-sheet.tsx` (`'use client'`): a
  `PhoneDialog variant="sheet" role="alertdialog"` per contracts/phone-ui.md, with a red
  filled "Delete spending" and a "Cancel". Add the strings "Delete this spending?" and
  "Deleted {merchant} · {amount}" to `messages.ts` · skill: frontend-developer:frontend-developer
- [X] T039 [US3] In `fe/app/ui/phone/phone-app.tsx`, open the form in edit mode for the tapped
  row's `spendingId`, and render `DeleteSheet` when `sheet === 'confirm-delete'`. Make T035 and
  T036 pass · skill: frontend-developer:frontend-developer

**Checkpoint**: All P1 stories work; this is shippable as the first phone release if wanted.

---

## Phase 6: User Story 4 — See the month by category on the phone (Priority: P2)

**Goal**: "By category" shows a Table | Pie toggle. The table lists every category with its
share and a Total; the pie shows the top 6 plus "Everything else".

**Independent Test**: In a month with 8 categories, the table lists 8 with a total equal to the
month total. The pie shows 6 slices plus "Everything else", equal to the other 2 (spec US4).

### Tests first

- [X] T040 [P] [US4] Extend `fe/app/lib/phone.test.ts` for
  `topCategories(items, totalCents, 6)`:
  - **8 categories**: 6 by rank plus `{ name: 'Everything else', cents: sum of the other 2 }`.
  - **6 or fewer**: no "Everything else".
  - **Shares**: from `sharePercent`.
  - **Colours**: `--color-cat-1` … `--color-cat-6` by rank, and `var(--color-muted)` for the rest.

  · skill: frontend-developer:frontend-developer
- [X] T041 [P] [US4] Create `fe/app/ui/phone/category-breakdown.test.tsx` covering:
  - **Toggle**: a "Show as" group with "Table" and "Pie" (`aria-pressed`) bound to `setChart`.
  - **Table**: a table "Spending by category" with "Category", "Amount" and "Share" columns,
    every category, and a "Total" row with the month total and "100%".
  - **Pie**: an `img` with the contract's label, the total in the middle, and a legend list.
  - **Empty month**: "Nothing to break down in [Month Year]."

  · skill: frontend-developer:frontend-developer
- [X] T042 [P] [US4] Create `fe/e2e/phone/by-category.spec.ts` covering US4 scenarios 1–6:
  the switch; table rows and total; pie legend with "Everything else" when there are more than
  6 categories; the chart choice kept after changing month; the empty text; and `checkScreen` ·
  skill: frontend-developer:frontend-developer

### Implementation

- [X] T043 [P] [US4] In `fe/app/lib/phone.ts`, add `topCategories` (R-013). Make T040 pass ·
  skill: frontend-developer:frontend-developer
- [X] T044 [US4] Create `fe/app/ui/phone/category-breakdown.tsx` (`'use client'`):
  - **Pie**: a donut from `donutArcs` (180 viewBox, about 200px), with the legend rows showing
    the swatch, name (truncate), amount and share.
  - **Table**: a full-width grid with a 3-column layout that fits 320px.

  In `fe/app/ui/phone/phone-app.tsx`, render it when `view === 'by-category'`. Make T041 and
  T042 pass · skill: frontend-developer:frontend-developer

---

## Phase 7: User Story 5 — Account menu, Settings and signing out on the phone (Priority: P2)

**Goal**: The user icon opens a bottom sheet with Settings, Import CSV and Sign out. Settings is
full screen with Categories | Accounts tabs and the same add and remove rules as the desktop.

**Independent Test**: Add "Coffee", get "coffee" refused, and remove "Coffee". Add an account,
see "No spendings yet", and remove it. Removing an account in use is refused. Sign out shows the
signed-out screen (spec US5).

### Tests first

- [X] T045 [P] [US5] Create `fe/e2e/phone/menu-settings.spec.ts` covering US5 scenarios 1–8.
  Seed via the API helpers the desktop `settings.spec.ts` uses, by copying, not importing.
  - **Menu**: the sheet's items; a backdrop tap closes it.
  - **Categories**: "Your categories · N"; add; a duplicate in any case gives
    "“Coffee” already exists."; empty gives "Type a name for the category."; a category in use
    can't be removed (message with the count); the 17 defaults have no remove button.
  - **Accounts**: usage text; the in-use and last-account refusals.
  - **Sign out**: it shows "You're signed out".
  - **Screen checks**: `checkScreen` on the menu and both tabs.

  · skill: frontend-developer:frontend-developer
- [X] T046 [P] [US5] Create unit tests for the US5 components:
  - `fe/app/ui/phone/account-menu.test.tsx`:
    - a dialog "Account menu" with buttons "Settings" and "Import CSV" and a link "Sign out"
      to `/auth/logout`;
    - "Close menu" on the backdrop;
    - "Settings" and "Import CSV" open their screens.
  - `fe/app/ui/phone/phone-settings.test.tsx`:
    - "Close settings"; the "Settings section" tabs; Categories by default;
    - the stacked "New category" input with "Add category" calling `createCategory` (mocked);
      the existing messages for empty, duplicate and in-use names;
    - the defaults as plain list items;
    - Accounts with "12 spendings recorded", "1 spending recorded" and "No spendings yet";
    - `deleteAccount` refusals shown as `role="alert"`.

  · skill: frontend-developer:frontend-developer

### Implementation

- [X] T047 [P] [US5] Create `fe/app/ui/phone/account-menu.tsx` (`'use client'`): a
  `PhoneDialog variant="sheet"` with full-width rows at least 52px tall for "Settings",
  "Import CSV", a divider, and a plain `<a href="/auth/logout">` "Sign out", which is not
  prefetched (as in `app-header.tsx`) · skill: frontend-developer:frontend-developer
- [X] T048 [P] [US5] Create `fe/app/ui/phone/phone-settings.tsx` (`'use client'`): a
  `PhoneDialog variant="screen"` labelled "Settings", built per contracts/phone-ui.md. It
  calls the existing `createCategory`, `deleteCategory`, `createAccount` and `deleteAccount`.
  - Check duplicates locally, ignoring case, against the defaults plus own categories (or the
    accounts) before calling, with the existing message. The server stays authoritative.
  - Add the explanatory texts and "No custom categories yet." to `messages.ts` (wording from
    the design).

  · skill: frontend-developer:frontend-developer
- [X] T049 [US5] In `fe/app/ui/phone/phone-app.tsx`, render `AccountMenu` when
  `sheet === 'menu'` and `PhoneSettings` when `screen === 'settings'`. Changes must refresh the
  form's suggestion lists without a page reload (the actions' `refresh()` re-renders
  `PhoneSection`; FR-085). Make T045 and T046 pass · skill:
  frontend-developer:frontend-developer

---

## Phase 8: User Story 6 — Import a CSV file on the phone (Priority: P3)

**Goal**: Choose a CSV file, review every row with the problems marked, fix or remove rows on
the "Fix row" screen, and add them all at once.

**Independent Test**: Import 10 rows, 4 of them with problems. The banner says "4 rows need
attention" and Add is disabled. Fix 3 rows and remove 1. Then "All rows look good" shows,
"Add 9 spendings" saves 9 and lands on the latest month with "Added 9 spendings from [file]"
(spec US6).

### Tests first

- [X] T050 [P] [US6] Add the fixture `fe/e2e/fixtures/phone-import.csv`. It has the header
  `Date,Description,Category,Account,Amount`, then 10 rows taken from the design's
  `SAMPLE_CSV`:
  - a quoted `"Lakeshore Fuel, Hwy 7"`;
  - an empty category and an empty account;
  - an empty date and an empty amount;
  - `"-18.90"`;
  - `10/27/2026`, unreadable (day first);
  - the new category "Coffee".

  Dates are within the last month, so none are in the future. Merchants carry the
  `e2e-phone-import` prefix for cleanup · skill: frontend-developer:frontend-developer
- [X] T051 [P] [US6] Create `fe/e2e/phone/import-csv.spec.ts` covering US6 scenarios 1–11:
  - **Pick errors**: `not-csv.png` gives the not-CSV message; `empty.csv` and
    `header-only.csv` give their messages.
  - **Review**:
    - the subtitle "phone-import.csv · 10 rows";
    - the banner count; the amber texts "No date", "No category", "No account" and
      "No amount";
    - `-18.90` shown as $18.90 and `"Lakeshore Fuel, Hwy 7"` as one merchant.
  - **Fix row**: "Save row" refuses an incomplete row with the inline message; fixing works;
    "Remove row" removes at once.
  - **Add**: it is disabled as "Add spendings" until all rows are valid; "Add 9 spendings"
    lands on the latest month with the notice and the new category created.
  - **Back**: the back button discards the review.
  - **Screen checks**: `checkScreen` on both screens.

  Clean up the imported spendings and "Coffee" afterwards · skill:
  frontend-developer:frontend-developer
- [X] T052 [P] [US6] Extend `fe/app/lib/phone.test.ts` for
  `importRowDisplay(row, today)` per data-model.md, "Derived per row":
  - the part texts, `needsAttention` and an amber flag per part;
  - a future date keeps its "Oct 27" text with the amber flag;
  - `importBanner(rows, today)`: "N rows need attention", "1 row needs attention" or
    "All rows look good";
  - `importAddLabel(rows, today)`: `{ label: 'Add spendings', disabled: true }` while blocked
    or empty, otherwise "Add N spendings" or "Add 1 spending".

  · skill: frontend-developer:frontend-developer
- [X] T053 [P] [US6] Create unit tests for the US6 components:
  - `fe/app/ui/phone/phone-import.test.tsx`:
    - "Close import"; the "Choose a CSV file" input with `accept=".csv,text/csv"`; no drop
      zone;
    - `checkFile` / `readImport` errors shown as an alert;
    - a good file opens review with the rows and a fresh idempotency key.
  - `fe/app/ui/phone/phone-review.test.tsx`:
    - "Back to file selection"; the banner; "Tap a row to check or fix it.";
    - row buttons with the contract's names; warning vs check icon;
    - "No rows left to import.";
    - add calls `importSpendings` once with the rows and the key, then
      `closeThen(router.push(month))` and `notify('Added …')`;
    - a failure keeps the rows and shows the message.
  - Add fix-row cases to `fe/app/ui/phone/spending-form.test.tsx`:
    - title "Fix row"; button "Save row"; delete label "Remove row" with no sheet;
    - an invalid row is not saved;
    - a valid save updates only the review row (no Server Action call).

  · skill: frontend-developer:frontend-developer

### Implementation

- [X] T054 [P] [US6] In `fe/app/lib/phone.ts`, add `importRowDisplay`, `importBanner` and
  `importAddLabel`, built on `draftProblems`. Add the strings to `messages.ts`: "Merchant
  missing", "No date", "No category", "No account", "No amount", "Tap a row to check or fix
  it.", "No rows left to import.", "All rows look good" and the plural forms. Make T052 pass ·
  skill: frontend-developer:frontend-developer
- [X] T055 [P] [US6] Create `fe/app/ui/phone/phone-import.tsx` (`'use client'`): the choose-file
  step per contracts/phone-ui.md. It uses `checkFile`, then `readImport` with the lookups. On
  success it stores `{ fileName, rows, idempotencyKey: crypto.randomUUID() }` in the provider's
  import session and calls `openScreen('review')` · skill: frontend-developer:frontend-developer
- [X] T056 [P] [US6] Create `fe/app/ui/phone/phone-review.tsx` (`'use client'`): the review
  list per contracts/phone-ui.md and data-model.md. Amber text uses a colour token that meets
  4.5:1 on the ground colour. The add bar is fixed at the bottom, with a summary of the row
  count. It calls `importSpendings` (reusing the session key on retry) · skill:
  frontend-developer:frontend-developer
- [X] T057 [US6] In `fe/app/ui/phone/spending-form.tsx`, implement `mode: 'fix-row'`:
  - title "Fix row"; "Save row";
  - validation as in add mode, with no saving to the server;
  - on a valid save, replace the row in the import session and close back to review;
  - "Remove row" removes the row and closes, with no confirmation.

  · skill: frontend-developer:frontend-developer
- [X] T058 [US6] In `fe/app/ui/phone/phone-app.tsx`, render `PhoneImport`, `PhoneReview` and the
  fix-row form for their screens, wiring "Import CSV" from the menu. Make T051 and T053 pass ·
  skill: frontend-developer:frontend-developer

---

## Phase 9: Polish & Cross-Cutting Concerns

- [X] T059 [P] Create `fe/e2e/phone/back-gesture.spec.ts` (FR-103, R-006):
  - On each of Edit, Settings, Import CSV, Review import, the menu and the delete sheet,
    `page.goBack()` closes only that layer, and Review goes back to Import CSV.
  - Back on the list leaves as the browser normally does.
  - A reload on an open screen shows the list.
  - After saving a spending in another month, one Back does not reopen the form.

  · skill: frontend-developer:frontend-developer
- [X] T060 [P] Create `fe/e2e/phone/a11y.spec.ts`: axe scans (`@axe-core/playwright`, rules as
  in the desktop `a11y.spec.ts`) on the list, By category Table and Pie, the menu, Edit, the
  delete sheet, the calendar, both Settings tabs, Import CSV and Review import. Expect no
  serious or critical violations (SC-006) · skill: frontend-developer:frontend-developer
- [X] T061 [P] Layout switch e2e in `fe/e2e/phone/layout-switch.spec.ts`:
  - At 390px, resize to 800px: the desktop month heading shows with the same `?month=`, and
    the phone root is gone. Resize back: the phone root shows.
  - With the Edit screen open, resize to 800px and back: the screen is closed and nothing is
    saved.

  · skill: frontend-developer:frontend-developer
- [X] T062 Run the full desktop regression (SC-005):
  1. `pnpm e2e --project=chromium` is green with the T001 count.
  2. `git diff origin/main --stat` lists no file under `fe/app/ui/spendings/`,
     `fe/app/ui/import/`, `fe/app/ui/settings/`, `fe/app/ui/app-header.tsx`, or any existing
     `fe/e2e/*.spec.ts` or `*.test.ts(x)`.
  3. Repeat the screenshots of T002 from the branch and compare them; they must match.

  Record the results in Implementation notes · skill: frontend-developer:frontend-developer
- [X] T063 Run the gates: in `fe/`, `pnpm typecheck && pnpm lint && pnpm test && pnpm e2e`
  (both projects). Fix anything red without touching desktop files · skill:
  frontend-developer:frontend-developer
- [X] T064 [P] Update `fe/README.md`: the phone layout (below 640px), where phone code lives,
  and how to run `pnpm e2e --project=phone` · skill: frontend-developer:frontend-developer
- [ ] T065 Run quickstart.md §3 by hand at 390px locally, including steps 8 (iPhone zoom) and 9
  (rotation) on a real phone once deployed. Note anything that differs from the spec in
  Implementation notes · skill: none
- [X] T066 Ship, with the user's agreement:
  1. Open the frontend PR and get CI green.
  2. Merge, which runs the release workflow, then merge the hl-helm PR. Check Argo CD is
     Synced/Healthy.
  3. Repeat quickstart §3 on a phone at https://budget.valyalkin.com.
  4. Update `specs/006-mobile-view/spec.md` Status, then add the Features row to the spec
     README via a small PR.

  · skill: homelab:hl-deploy-status

---

## Implementation notes

(2026-10-31, frontend branch `006-mobile-view`: commits `f76f23e`, `afa7104`, `2b186de`. No backend change.)

- **Desktop baseline (T001)**: on unmodified `origin/main`, without E2E credentials, 7 passed and
  29 skipped. With credentials, the branch passes all 35 desktop journeys; 1 is skipped by design
  (the API-down case needs a second server).
- **Desktop proof (T002, T062)**: the baseline screenshots were taken at the end, not the start:
  the same signed-in month, entry row, Settings and Import CSV windows at 1280px and 1440px from
  `origin/main` (a temporary checkout) and from the branch. All 8 pairs are byte-for-byte
  identical. `git diff origin/main` touches no file under `app/ui/spendings/`, `app/ui/import/`,
  `app/ui/settings/`, `app/ui/app-header.tsx`, and no existing `*.test.ts(x)` or `e2e/*.spec.ts`.
- **Gates**: `pnpm typecheck && pnpm lint && pnpm test`: 52 files, 429 tests (96 new). Phone e2e:
  27 passed. Desktop e2e: 35 passed.
- **Row names (deviation from contracts/phone-ui.md)**: axe's `label-content-name-mismatch`
  flagged `aria-label` on list and review rows. Rows now take their name from content, with
  visually hidden extras: the list reads "Oct 27, Groceries, $84.37, Neighbourhood Market" (the
  merchant last), and a review row reads "merchant, date · category · account, amount" plus
  ", needs attention". The contract is updated.
- **"Everything else" colour**: `--color-muted` (there is no `--color-ink-3` token). data-model.md
  is updated.
- **New tokens**: `--color-warn` and `--color-warn-tint` (amber for the review), added in
  `globals.css`; unused by the desktop.
- **Earliest-year test (T008)**: lives in a new `app/lib/data-earliest.test.ts`, so no existing test
  file is edited. React `cache` deduplication cannot be observed in Vitest (it is a pass-through
  outside a server render), so it is not asserted there.
- **Import fixture (T050)**: built at run time in `e2e/phone/import-csv.spec.ts` instead of a
  committed `phone-import.csv`, so its dates never go stale (the desktop import spec does the same).
- **Signed-out screen (T026)**: already fits 320px with a 44px Sign in; no change was needed.
- **Double taps**: the form, delete sheet and review stay locked after a successful save until they
  close, so a second tap does nothing.
- **Auth0 rate limit**: running both e2e projects in one go (about 60 sign-ins) timed out four tests
  on the login page; `pnpm e2e --last-failed` passed them. The frontend README notes this.
- **Test data**: an early failed phone run left marked spendings and a category; a one-off cleanup
  removed them. The test account still holds older desktop-run data (for example
  `Cat e2e-add-…` categories and `e2e-import-fix-…` merchants), which shows up in suggestions.
- **Deploy (T066)**: hl-budget-frontend#7 merged (gitleaks first flagged a made-up UUID in a test
  as an API key; replaced by `crypto.randomUUID()` and listed in `.gitleaksignore`). Release
  `17-ac055d4`, hl-helm#26 merged, Argo CD Synced/Healthy. The 27 phone journeys pass against
  https://budget.valyalkin.com.
- **Open**: T065, by hand on a real iPhone (zoom, keyboard, back swipe, rotation).

---

## Dependencies & Execution Order

### Phase dependencies

- **Setup (T001–T002)** comes first.
- **Foundational (T003–T019)** depends on Setup and blocks every story.
  - Tests T003–T008 run in parallel.
  - T009–T012 run in parallel.
  - T013 needs T012.
  - T015 needs T013 and T014.
  - T016 needs T011 and T015.
  - T017–T019 run in parallel with T013–T016.
- **US1 (T020–T026)** needs Foundational. It is the MVP and the base of the main screen.
- **US2 (T027–T034)** needs US1 (the floating button and the app shell).
- **US3 (T035–T039)** needs US2 (`spending-form.tsx`).
- **US4 (T040–T044)** needs US1 only. It can run in parallel with US2 and US3.
- **US5 (T045–T049)** needs US1 only. It can run in parallel with US2–US4.
- **US6 (T050–T058)** needs US2 (form, fix-row mode), and US5 for the "Import CSV" entry in the
  menu.
- **Polish (T059–T066)** comes after the stories it covers. T059 needs US3, US5 and US6;
  T062–T066 come last.

### Shared-file ordering

`fe/app/ui/phone/phone-app.tsx`, `spending-form.tsx`, `fe/app/lib/phone.ts` and
`fe/app/lib/messages.ts` are touched by several stories. Tasks that edit the same file are
**not** marked [P] against each other. Run them in story order, or merge carefully when
stories run in parallel.

## Parallel Example: Foundational tests

```text
T003 use-is-phone.test.ts   T004 layout-switch.test.tsx   T005 phone-ui-provider.test.tsx
T006 phone-dialog.test.tsx  T007 phone.test.ts            T008 data.test.ts (new cases)
```

## Parallel Example: User Story 1

```text
Tests:  T020 e2e/phone/month-view.spec.ts  |  T021 header/picker/list unit tests
Build:  T022 phone-header.tsx  |  T023 month-year-picker.tsx  |  T024 spending-list.tsx
Then:   T025 compose in phone-app.tsx → T026 signed-out max-sm: tweaks
```

## Parallel Example: after US1

```text
Track A: US2 → US3 (form, delete)
Track B: US4 (breakdown)
Track C: US5 (menu, settings)
then US6 (needs A and C)
```

## Implementation Strategy

1. **MVP**: Setup, then Foundational, then US1. The phone shows the month correctly, and the
   desktop is proven unchanged (T016 check plus the desktop e2e run).
2. **First useful release**: add US2 and US3, so the user can view, add, edit and delete on the
   phone. Stopping here is a valid release.
3. **Complete**: add US4, then US5, then US6, each checked with its own e2e spec before moving
   on.
4. **Before merging**: the Polish phase, especially T062 (the desktop regression proof) and
   T059 (back gesture, the riskiest decision, R-006). Consider pulling T059 forward to right
   after US2 to confirm the history approach early.
