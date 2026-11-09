---

description: "Task list for 010-quick-add-and-undo"
---

# Tasks: Quick add from a date, and undo for delete

**Input**: Design documents from `/specs/010-quick-add-and-undo/`

**Prerequisites**: plan.md, spec.md, research.md, data-model.md, contracts/ui-entry-table.md,
quickstart.md

**Tests**: REQUIRED. Constitution Principle IV says tests ship with the code and every new
user-visible flow gets a Playwright spec. In each phase, write the tests first and see them fail
before writing the code that makes them pass.

**Organization**: Tasks follow spec.md US1 (quick add) and US2 (undo for delete), both P1 and
independent of each other. All code is in the frontend repository `hl-budget-frontend` (written
`fe/` below). There is no backend work. The table component is shared by Spendings and Income
through `useLedger()`, so each change covers both tabs, and each e2e covers both tabs.

**Branch**: create `010-quick-add-and-undo` in `fe/` from an up-to-date `origin/main` (local
`main` is behind).

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Can run in parallel (different files, no dependency on unfinished tasks)
- **[Story]**: US1 or US2 (spec.md)

## Path Conventions

- Frontend: `fe/app/...`, `fe/e2e/...` (`fe/` = `/Users/valyalkin/Documents/Projects/hl-budget-frontend`)
- Spec docs: `specs/010-quick-add-and-undo/...`

---

## Phase 1: Setup

- [X] T001 In `fe/`, run `git fetch && git switch -c 010-quick-add-and-undo origin/main`, then
  `pnpm install`, then run `pnpm typecheck && pnpm lint && pnpm test` and check they are green
  before any change. Read `fe/app/ui/spendings/spendings-table.tsx`,
  `fe/app/ui/spendings/spending-row.tsx`, `fe/app/ui/spendings/entry-row.tsx`,
  `fe/app/ui/spendings/date-picker.tsx` and `fe/app/lib/actions.ts` (`deleteSpending`,
  `attempt`) to confirm the shapes described in plan.md "Design Overview".

---

## Phase 2: Foundational

No shared prerequisites. US1 and US2 touch different parts of the same two components
(`spendings-table.tsx`, `spending-row.tsx`). If both are built in parallel, merge carefully, or
build them one after the other (US1 first, because it is smaller).

---

## Phase 3: User Story 1 — Add an entry for a day already in the table (Priority: P1) 🎯 MVP

**Goal**: A saved row's date is a button "Add on {date}". It opens the add row with that date,
the calendar closed and focus in the merchant/party field.

**Independent Test**: On October, with an entry on Oct 14, click "Oct 14". The new row opens
with date Oct 14, the calendar closed and focus in merchant. Fill in the rest, save, and check
the entry is listed under Oct 14 on the same month. Repeat on Income.

### Tests for User Story 1 ⚠️ (write first, see them fail)

- [X] T002 [P] [US1] Add `fe/app/ui/spendings/spending-row.test.tsx`. Cases:
  - With `locked={false}` and not pending, the date renders as
    `getByRole('button', { name: 'Add on Oct 14' })` with `title="Add on Oct 14"`. Clicking it,
    or pressing Enter or Space on it, calls `onAddOnDate` once.
  - With `locked={true}`, there is no button with that name and the date is plain text.
  - Edit and remove behave as today.
- [X] T003 [P] [US1] Extend `fe/app/ui/spendings/entry-row.test.tsx`:
  - With `mode="add"`, `openCalendar={false}` and `startField="merchant"`, the calendar popup
    is not rendered and `document.activeElement` is the merchant/party combobox input after
    mount.
  - Without these props, add mode still opens the calendar (regression).
  - The date button shows the given `initial.date`, and opening it shows that month with the
    day selected.
- [X] T004 [P] [US1] Extend `fe/app/ui/spendings/spendings-table.test.tsx`:
  - Clicking "Add on Oct 14" on a row dated `2026-10-14` renders exactly one add row at the top
    of the body with date `2026-10-14`.
  - While an add or edit row is open, no "Add on …" buttons exist.
  - Cancelling (Escape) closes the row and the date buttons come back.
  - Run the case once with the Income ledger context (party label "Source").
- [X] T005 [P] [US1] Add `fe/e2e/quick-add-undo.spec.ts` with `test.describe('US1: quick add from a date')`
  for both Spendings (`/spendings`) and Income (`/income`). Use the helpers in `fe/e2e/support.ts`
  and `fe/e2e/income/support.ts` (marker, add via UI, cleanup). Cases:
  - **Mouse**: hover the date and assert `title` is "Add on {date}", click it, assert the new
    row's date button text equals the clicked date, assert no calendar is open, assert the
    party field is focused, fill category, account and amount, press Enter, and assert two rows
    with that date and the URL month unchanged.
  - **Keyboard**: focus the date button, press Enter, and get the same result.
  - **Locked**: open + and assert `getByRole('button', { name: /^Add on / })` has count 0.

### Implementation for User Story 1

- [X] T006 [US1] In `fe/app/ui/spendings/entry-row.tsx`, add two optional props to
  `EntryRowProps`:
  - `openCalendar?: boolean`, defaulting to `mode === 'add'`, passed to
    `<DatePicker autoOpen={…}>` in place of the current `autoOpen={mode === 'add'}`.
  - `startField?: Field`. When set, focus that field once on mount with a `useEffect` that runs
    on mount only, using `refs.current[startField]`.

  Document both in the component's comment (spec 010 FR-002, research R7). This makes T003 pass.
- [X] T007 [US1] In `fe/app/ui/spendings/spending-row.tsx`:
  - Add props `onAddOnDate: () => void` and `pending?: boolean`. The `pending` prop is only
    read here and US2 uses it. Default `false`.
  - Render the date cell content as `<button type="button" aria-label={`Add on ${shortDate(date)}`}
    title={…same…} onClick={onAddOnDate}>` when `!locked && !pending`, and as plain text
    otherwise (research R8).
  - Style the button so the cell looks as it does today plus a pointer cursor, an underline on
    hover or focus-visible, and the existing focus-visible outline classes.

  This makes T002 pass.
- [X] T008 [US1] In `fe/app/ui/spendings/spendings-table.tsx`:
  - Change the `OpenRow` type to
    `null | { mode: 'add'; date?: string } | { mode: 'edit'; spendingId: string }`
    (data-model.md).
  - Pass `onAddOnDate={() => { setMessage(''); setOpen({ mode: 'add', date: spending.date }); }}`
    to each `SpendingRow`.
  - Render the add `EntryRow` with `initial.date = open.date ?? defaultDateFor(ym, today)`,
    `openCalendar={!open.date}` and `startField={open.date ? 'merchant' : undefined}`.
  - Keep `key` stable so a second quick add after cancel remounts with the new date. Add
    `key={open.date ?? 'add'}` to the add row.

  This makes T004 pass.
- [X] T009 [US1] Run `pnpm typecheck && pnpm lint && pnpm test` and
  `pnpm exec playwright test e2e/quick-add-undo.spec.ts -g "US1"`. Fix until green. Check that
  `e2e/add-spending.spec.ts`, `e2e/keyboard-autofill.spec.ts`, `e2e/a11y.spec.ts` and
  `e2e/income/add-edit-delete.spec.ts` still pass, because the date cell changed role.

**Checkpoint**: US1 works on both tabs and is shippable on its own.

---

## Phase 4: User Story 2 — Undo an accidental delete (Priority: P1)

**Goal**: Remove fades the row and shows an always-visible Undo for 5 s. After that the existing
delete runs exactly once. Leaving within the app runs pending deletes at once, and a page unload
keeps them.

**Independent Test**: Remove a row, click Undo, reload, and check it is still there. Remove
again, wait 5 s, and check the row disappears and the totals drop. Reload and check it is gone.
Repeat on Income.

### Tests for User Story 2 ⚠️ (write first, see them fail)

- [X] T010 [P] [US2] Add `fe/app/ui/spendings/use-pending-deletes.test.ts` (Vitest +
  `renderHook`, `vi.useFakeTimers()`). Cases:
  - `start(id)` makes `phaseOf(id) === 'waiting'`. After 4 999 ms `onDelete` has not been
    called. After 1 more ms it has been called once with `id` and the phase is `'sending'`.
  - `undo(id)` while waiting clears it, and `onDelete` is never called even after 10 s.
  - `undo` while sending is a no-op.
  - `start` on an already pending id is ignored.
  - Two ids started 1 s apart fire 1 s apart.
  - `flushAll()` sends every waiting id at once and they are not sent again when their timers
    would have fired.
  - Unmount calls `flushAll()`.
  - Dispatching `window` event `hl-budget:flush-deletes` flushes.
  - Results:
    - `onDelete` resolving `{ status: 'error', message }` drops the id and calls
      `onError(message)`.
    - `{ status: 'error', gone: true }` is treated like `ok` with no `onError`.
    - `{ status: 'signed-out' }` calls `onSignedOut`.
  - `settle(ids)` drops `sending` ids that are no longer in the given list (research R6).
  - Strict Mode double effects do not cause double sends (research R4).
- [X] T011 [P] [US2] Extend `fe/app/ui/spendings/spending-row.test.tsx`:
  - With `pending` set to `'waiting'`, the row has `data-pending="true"` and an opacity class,
    there are no Edit or Remove buttons, and `getByRole('button', { name: 'Undo' })` (title
    "Undo remove") is visible without hover. Clicking it calls `onUndo`.
  - With `pending` set to `'sending'`, Undo is disabled.
  - In both pending states the date is plain text.

  (Widen the US1 boolean `pending` prop to `'waiting' | 'sending' | undefined` as part of T015.)
- [X] T012 [P] [US2] Extend `fe/app/ui/spendings/spendings-table.test.tsx` (mock
  `deleteSpending`, fake timers):
  - Remove makes the row pending and moves focus to its Undo. The status region (`role="status"`)
    reads "{party} on {date} removed. Undo available for 5 seconds." `deleteSpending` has not
    been called yet.
  - Undo restores the row, focuses its Remove button and announces "{party} on {date} restored.".
  - After 5 s `deleteSpending` is called once with `(id, ledger)`.
  - On an `error` result the row is normal and the existing error line shows the message.
  - Two rows can be pending at once.
  - The footer total is unchanged while pending.
  - Unmounting the table sends pending deletes.
- [X] T013 [P] [US2] Add `test.describe('US2: undo for delete')` to `fe/e2e/quick-add-undo.spec.ts`
  for both Spendings and Income. Use `await page.clock.install()` before navigation and
  `page.clock.runFor(5000)` to pass the wait (research R9). Cases:
  - **Undo**: remove, then assert `data-pending="true"` and Undo focused. Press Enter, reload,
    and assert the row exists.
  - **Delete after wait**: remove, then assert the footer total is unchanged. Run 5 000 ms,
    then assert the row is gone and the footer total dropped by the amount. Reload and assert
    it is still gone.
  - **Several**: remove three rows, undo the middle one, run 5 000 ms, and assert only the
    middle one remains.
  - **Leave flushes**: remove, switch to the previous month within 5 s, come back, and assert
    the row is gone. Repeat by opening Settings.
  - **Unload keeps**: remove and `page.reload()` within 5 s, then assert the row is still
    there.
  - **One real-clock smoke test**: remove, wait for the row to disappear with a 7 s timeout.
- [X] T014 [P] [US2] Update the existing e2e tests that expect an immediate delete. Make them
  click Remove and then run the clock or wait for the row to disappear (up to 7 s):
  - `fe/e2e/edit-delete.spec.ts`: rename "remove it at once" accordingly.
  - `fe/e2e/income/add-edit-delete.spec.ts`
  - `fe/e2e/failures.spec.ts` (the "not deleted" case now fails after the wait and the row
    returns to normal).
  - `fe/e2e/settings.spec.ts` and `fe/e2e/income/settings.spec.ts`, wherever they delete an
    entry before removing a category or account.

  Do not change `fe/e2e/phone/*` (SC-005).

### Implementation for User Story 2

- [X] T015 [US2] Add `fe/app/lib/pending-deletes.ts`, exporting:
  - `FLUSH_DELETES_EVENT = 'hl-budget:flush-deletes'`
  - `flushPendingDeletes(): void`, which dispatches a `CustomEvent` on `window`

  Follow the pattern in `fe/app/ui/settings/open-settings.ts`.
- [X] T016 [US2] Add `fe/app/ui/spendings/use-pending-deletes.ts` exporting
  `usePendingDeletes({ onDelete, onDeleted, onError, onSignedOut })`. The callbacks are:
  - `onDelete(id): Promise<ActionResult>`
  - `onDeleted(id)`
  - `onError(id, message)`
  - `onSignedOut()`

  It returns `{ phaseOf(id), start(id), undo(id), flushAll(), settle(listedIds) }`.

  Rules (data-model.md "Pending delete"):
  - State is `Map<string, 'waiting' | 'sending'>`. Timers are kept in a ref.
  - The delay is `UNDO_MS = 5000`, exported.
  - A `sent` ref Set guarantees "An id moves to `sending` at most once per page".
  - "`undo` is allowed only in `waiting`".
  - "`start` is ignored if the id is already pending".
  - A `gone: true` error counts as success.
  - Keep callbacks in refs so timers call the latest ones.
  - The unmount cleanup clears timers and calls `flushAll()`. Put the flush inside cleanup
    with a ref guard so Strict Mode's simulated unmount does not send. Do this by deferring the
    flush with `queueMicrotask` and cancelling it if the effect re-runs.
  - Subscribe to `FLUSH_DELETES_EVENT`.

  This makes T010 pass.
- [X] T017 [US2] In `fe/app/ui/spendings/spending-row.tsx`:
  - Widen `pending` to `'waiting' | 'sending' | undefined` and add `onUndo: () => void` plus
    `undoRef` / `removeRef` callback refs, or `forwardRef`-style props, so the table can move
    focus.
  - When pending, set `data-pending="true"` on the `<tr>` and `opacity-50` on the data cells.
    Replace the Edit and Remove buttons with one always-visible Undo button. "Always visible"
    means it is not inside the `opacity-0 group-hover` wrapper. The Undo button has
    `aria-label="Undo"`, `title="Undo remove"`, an undo-arrow SVG with `aria-hidden` in the
    same 16×16 style as the other icons, the same `actionClass`, and
    `disabled={pending === 'sending'}`.
  - Remove the old `removing` prop.

  This makes T011 pass.
- [X] T018 [US2] In `fe/app/ui/spendings/spendings-table.tsx`:
  - Replace the `removing` state and the `remove()` function with `usePendingDeletes`, where
    `onDelete = (id) => deleteSpending(id, ledger)`.
    - `onError` sets the existing `message` and announces nothing extra.
    - `onSignedOut` calls `router.replace('/')`.
    - `onDeleted` announces "{party} on {date} deleted.".
  - Add a visually hidden `<p role="status" aria-live="polite" aria-atomic="true"
    className="sr-only">` for the remove, restore and delete announcements
    (contract "Status announcements"). Keep the existing error `<p>` unchanged.
  - On remove:
    - Call `start(id)`.
    - Announce "{merchant} on {shortDate(date)} removed. Undo available for 5 seconds.".
    - Focus that row's Undo after render, using a `focusAfterRender` ref and `useEffect`.
  - On undo:
    - Call `undo(id)`.
    - Announce "… restored.".
    - Focus the row's Remove button.
  - Call `settle(spendings.map((s) => s.id))` in an effect on `spendings`.
  - Pass `pending={phaseOf(id)}` to each row.
  - A pending row's edit, remove and quick-add stay unavailable. Other rows are unaffected,
    and `locked` still comes from `open !== null`. Undo stays enabled while a row is open.

  This makes T012 pass.
- [X] T019 [P] [US2] In `fe/app/ui/settings/settings-dialog.tsx`, call `flushPendingDeletes()`
  in the open path (`openDialog` and the open-settings event handler) before `showModal()`.
- [X] T020 [P] [US2] In `fe/app/ui/import/import-dialog.tsx`, call `flushPendingDeletes()` in
  the effect that calls `showModal()`.
- [X] T021 [US2] Update the doc comments that say delete happens "at once": the
  `SpendingsTable` header comment and any `FR-060` references in
  `fe/app/ui/spendings/spending-row.tsx` and `fe/app/ui/spendings/spendings-table.tsx`. Point
  them to spec 010 FR-010–FR-019.
- [X] T022 [US2] Run `pnpm typecheck && pnpm lint && pnpm test` and
  `pnpm exec playwright test e2e/quick-add-undo.spec.ts e2e/edit-delete.spec.ts e2e/failures.spec.ts e2e/settings.spec.ts e2e/income`.
  Fix until green.

  > Done 2026-11-08 for typecheck, lint, all unit tests (543) and the production build. The
  > Playwright specs were written but not run locally, because this machine has no Auth0 test
  > user (`E2E_USERNAME` / `E2E_PASSWORD`), so they skip. They run in CI when `E2E_ENABLED` is
  > set. The US2 e2e uses real 5-second waits instead of `page.clock`, because faking the clock
  > also fakes `Date` and risks the Auth0 login (deviation from research R9).

**Checkpoint**: US1 and US2 both work on both tabs.

---

## Phase 5: Polish & Cross-Cutting Concerns

- [X] T023 [P] Update `fe/README.md` where it describes the table's add and delete behaviour
  (if it does): quick add from a date, and the 5-second undo.
- [X] T024 Run the full frontend gate: `pnpm typecheck && pnpm lint && pnpm test`, the full
  `pnpm exec playwright test` including `e2e/phone` and `e2e/a11y.spec.ts`, and
  `docker build` under the hardened constraints as in earlier features. All must be green
  (Principle IV, SC-004, SC-005).
- [X] T025 Walk through `specs/010-quick-add-and-undo/quickstart.md` manual checks 1–10 against
  the local stack on both tabs.
- [X] T026 Frontend release. Open the PR (Conventional Commit `feat:`) and merge it after the
  user agrees. Let the release workflow open the hl-helm PR, and merge it after the user agrees.
  Then verify with `homelab:hl-deploy-status`. There is no backend release.
- [X] T027 Live check at `https://budget.valyalkin.com` (desktop): quick add and Undo on both
  tabs. Then update the README Features table in the spec repo with a small PR (row for 010)
  and mark these tasks done.

---

## Dependencies & Execution Order

### Phase Dependencies

- Setup (T001) → US1 (T002–T009) and US2 (T010–T022) → Polish (T023–T027).
- There is no foundational phase.

### User Story Dependencies

- **US1** and **US2** are functionally independent; either can ship alone.
- They edit the same files (`spending-row.tsx`, `spendings-table.tsx`, their tests and
  `quick-add-undo.spec.ts`). One developer should do US1 first and then US2. US2's row work
  assumes US1's `pending`-aware date cell (T007), so if US2 is done first, it adds the plain-text
  date for pending rows itself.

### Within Each User Story

- Tests first, failing (T002–T005, T010–T014).
- Then lower layers before the table: `entry-row` / `spending-row` (T006–T007), then the table
  (T008). For US2: the event module (T015), then the hook (T016), then the row (T017), then the
  table (T018), then the dialogs (T019–T020).
- Run the story's gate last (T009, T022).

### Parallel Opportunities

- US1 tests T002, T003, T004 and T005 are in different files and can run in parallel.
- US2 tests T010, T011, T012, T013 and T014 are in different files and can run in parallel.
- T019 and T020 (two dialogs) can run in parallel with each other once T015 exists.
- T023 can run in parallel with T024.

---

## Parallel Example: User Story 2

```bash
# Tests together (different files):
Task: "T010 use-pending-deletes.test.ts"
Task: "T011 spending-row.test.tsx pending cases"
Task: "T012 spendings-table.test.tsx undo cases"
Task: "T013 e2e/quick-add-undo.spec.ts US2"
Task: "T014 update existing delete e2e specs"

# After T015, the two dialog hooks together:
Task: "T019 settings-dialog.tsx flush on open"
Task: "T020 import-dialog.tsx flush on open"
```

---

## Implementation Strategy

### MVP First (User Story 1 Only)

1. T001 setup.
2. US1 (T002–T009), then stop and validate on both tabs.
3. It can be released alone if wanted. It is smaller and lower risk.

### Incremental Delivery

1. US1, then gate.
2. US2, then gate. This is the larger change: timers, flush and announcements.
3. Polish, then one frontend release covering both stories (default), or two releases if US1
   ships first.

---

## Notes

- No backend, OpenAPI or migration work. Do not regenerate API types.
- The phone UI (`fe/app/ui/phone/*`, `fe/e2e/phone/*`) must not change.
- Pending deletes are never stored. A reload or sign-out within 5 s keeps the entry, by design
  (spec FR-018).
- Commit after each task or logical group with Conventional Commits.
