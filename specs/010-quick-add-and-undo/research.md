# Research: Quick add from a date, and undo for delete

No open `NEEDS CLARIFICATION` items came out of Technical Context. The decisions below settle
the design choices, based on the current frontend (`hl-budget-frontend`, origin/main).

## R1. Where the 5-second wait lives

- **Decision**: In the browser. The delete request is sent only when the timer ends.
- **Rationale**: Spec FR-021 says the server is unchanged. A browser timer needs no new
  endpoint, no soft-delete column and no restore path, and it satisfies Principles II and VI. The
  failure mode on a lost page is that the entry is kept, which the spec chose as the safer
  mistake.
- **Alternatives considered**: (a) A server soft delete plus a restore endpoint would be durable
  across page loads, but it needs a contract change, a migration and a purge job, which is
  overkill for one user. (b) Deleting at once and undoing by re-creating the entry would give
  the entry a new id and creation time, so it would sort differently among same-day entries.
  It also needs a POST from the client with full data.

## R2. Detecting "leaving the screen"

- **Decision**: Run the pending deletes when the table unmounts, and when the
  `hl-budget:flush-deletes` window event fires. The Settings and Import dialogs dispatch that
  event when they open.
- **Rationale**: `MonthSection` renders `<SpendingsTable key={year-month}>`, so a month or year
  change remounts the table, and Spendings ↔ Income is a route change. Settings and Import are
  dialogs over the same page, so they need an explicit signal. Settings matters because removing
  a category or account still used by a pending entry would otherwise be refused. This follows
  the existing `hl-budget:open-settings` window-event pattern (`app/ui/settings/open-settings.ts`).
- **Alternatives considered**: Intercepting router navigation means Next has no stable API for
  it, and wrapping every link is brittle. A shared React context provider would be heavier than
  one event for two callers.

## R3. Page unload and Sign out

- **Decision**: Do nothing on `pagehide` or `beforeunload`. The timers die and the entries are
  kept.
- **Rationale**: Sign out is a plain `<a href="/auth/logout">`, which is a full page load.
  Sending a delete during unload is unreliable, because server actions are fetches that the
  browser may cancel, and `sendBeacon` cannot carry the server-side token. The spec was updated:
  sign out now keeps pending entries.
- **Alternatives considered**: A `beforeunload` "Leave site?" prompt is annoying for a 5 s window
  and is not asked for.

## R4. Sending each delete exactly once

- **Decision**: The hook keeps a `sent` ref set of ids. Timer expiry and `flushAll()` both check
  and add to it before calling `onDelete`. Undo is refused once an id is `sending`.
- **Rationale**: Covers FR-017, including React Strict Mode running effects twice in dev and a
  flush racing an expiring timer.

## R5. Treating "already gone" (404) as success

- **Decision**: If `deleteSpending` returns `{ status: 'error', gone: true }`, treat it like `ok`
  and show no message.
- **Rationale**: `attempt()` already marks 404s with `gone: true` and refreshes, and spec FR-015
  asks for no error in that case.

## R6. Keeping the row faded until it disappears

- **Decision**: After a successful send, keep the id in the hook as `sending`. Drop it when the
  `spendings` prop no longer contains it (an effect compares ids) or when the action resolves
  without revalidation.
- **Rationale**: The server action's revalidated RSC payload and its return value arrive together,
  but clearing pending state first could flash the row back to full opacity for a frame.

## R7. Quick add: calendar closed, focus to merchant

- **Decision**: `EntryRow` gets `openCalendar?: boolean` (default: `mode === 'add'`, as today) and
  `startField?: Field`. When `startField` is set, the row focuses that field on mount.
- **Rationale**: The user picked the day by clicking it, so reopening the calendar would add a
  step. Today `DatePicker` already accepts `autoOpen`, so this only threads a value through.

## R8. Date cell semantics

- **Decision**: The date is a `<button type="button">` with visible text such as "Oct 14", an
  `aria-label` and `title` of "Add on Oct 14", a pointer cursor and an underline on hover or
  focus. While the row is locked or pending it is plain text, not a disabled button.
- **Rationale**: A real button gives Tab, Enter and Space for free and satisfies `jsx-a11y`.
  Rendering plain text while unavailable matches the spec's "must not look clickable" and keeps
  the Tab order short.

## R9. Tests with timers

- **Decision**: Unit tests use `vi.useFakeTimers()` and advance by 4 999 ms and then 1 ms.
  Playwright uses `page.clock.install()` for the 5 s wait, so the e2e tests stay fast and
  deterministic. One e2e test uses the real clock as a smoke check.
- **Rationale**: Playwright ≥1.45 supports `page.clock`, and the repo is on 1.64.
