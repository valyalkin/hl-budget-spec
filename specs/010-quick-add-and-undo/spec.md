# Feature Specification: Quick add from a date, and undo for delete

**Feature Branch**: `010-quick-add-and-undo`

**Created**: 2026-11-08

**Status**: Draft

## Clarifications

### Session 2026-11-08 (planning)

- Sign out leaves the page (a full page load), so it is treated like closing the page: pending
  entries are kept, not deleted. Edge Cases and FR-018 updated.

**Input**: User description: "I would like to add a behaviour to spendings table. Now, when I click on the date on the table, I would like to open the new spendings input row, but the date should be pre-set to the date just clicked on the calendar. Also, when the spending is deleted, I would like the "undo" operation. So once i hit "delete" it changes opacity and stays there for 5 seconds and then deletes, but it has an undo icon."

## Overview

Two small changes to the desktop table of a month's entries:

1. **Quick add from a date.** Clicking the date of any row opens the new-entry row with that
   date already filled in, so several entries for the same day are fast to record.
2. **Undo for delete.** Remove no longer deletes at once. The row fades, shows an Undo button,
   and is deleted 5 seconds later unless the user clicks Undo.

The Income tab (feature 009) uses the same table and the same mechanics as Spendings, so both
changes apply to the Spendings and Income tables alike. In this spec, "entry" means a spending
on the Spendings tab and an income entry on the Income tab.

This changes two earlier decisions in feature 003: "Remove deletes the spending at once, with no
confirmation and no undo" (FR-060 and the One-click delete assumption). The phone view (feature
006) is not changed: it keeps its delete confirmation sheet and has no date to click.

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Add an entry for a day already in the table (Priority: P1)

The user is entering a receipt pile. A row for Oct 14 is in the table. They click "Oct 14" in
that row and the new-entry row opens at the top of the table with the date set to Oct 14 and the
cursor in the merchant field. They type the rest and save as usual.

**Why this priority**: Recording several entries for the same day is the most common data-entry
pattern; today each one needs the calendar to be reopened and the day picked again.

**Independent Test**: Show October with a spending on Oct 14, click "Oct 14" in its row, check
the new row opens with date Oct 14 and focus in merchant, fill in the rest, save, and check the
new spending is listed under Oct 14.

**Acceptance Scenarios**:

1. **Given** the month's table with no row open, **When** the user clicks the date of a row,
   **Then** the new-entry row opens at the top of the table with that row's date filled in, the
   calendar closed, and focus in the merchant field.
2. **Given** the new row opened from a date, **When** the user opens its date field, **Then** the
   calendar shows the month of the filled-in date with that day selected, and the user can
   change it like any new row.
3. **Given** the new row opened from a date, **When** the user saves, Escapes or cancels, **Then**
   it behaves exactly like a new row opened with +: the same validation, save, cancel and
   keyboard rules apply.
4. **Given** a row is already open for adding or editing, **When** the user looks at or clicks the
   dates of other rows, **Then** nothing happens and the dates do not look clickable, the same
   way edit and remove are unavailable while a row is open.
5. **Given** the table, **When** the user moves the pointer over a row's date, **Then** the date
   shows it is clickable (pointer cursor and a hover style) and a tooltip reads "Add on Oct 14"
   (with that row's date).
6. **Given** a keyboard user, **When** they Tab through a row, **Then** the date can be focused and
   Enter or Space opens the new row with that date, and screen readers announce it as
   "Add on Oct 14" (with that row's date).

---

### User Story 2 - Undo an accidental delete (Priority: P1)

The user clicks remove on the wrong row. The row stays in place, fades to half opacity, and its
edit and remove buttons are replaced by an Undo button that stays visible without hovering.
They click Undo within 5 seconds and the row returns to normal, nothing deleted. If they leave
it, the entry is deleted when the 5 seconds are up.

**Why this priority**: Remove is a single click next to edit; a misclick today loses the entry
and it has to be typed in again.

**Independent Test**: Click remove on a spending, check the row fades and shows Undo, click Undo,
reload the page and check the spending is still there. Then click remove again, wait 5 seconds,
check the row disappears and the totals drop, reload and check it is gone.

**Acceptance Scenarios**:

1. **Given** a row, **When** the user clicks remove, **Then** the row stays in its place at
   reduced opacity (about half), its edit and remove buttons are replaced by an Undo button
   (an undo arrow icon labelled "Undo") that is visible without hovering, and focus moves to
   that Undo button.
2. **Given** a fading row, **When** the user clicks Undo (or presses Enter or Space on it) within
   5 seconds, **Then** the row returns to normal at full opacity, nothing is deleted, and focus
   returns to the row's remove button.
3. **Given** a fading row, **When** 5 seconds pass without Undo, **Then** the entry is deleted,
   the row leaves the table, and the footer, month line, total card and breakdown cards update.
4. **Given** a fading row, **When** the user looks at the totals, **Then** they still include the
   entry, because it is not deleted yet; they change only when the delete is carried out.
5. **Given** the delete is carried out after 5 seconds and fails, **When** the result comes back,
   **Then** the row returns to normal and the user sees the same short message as today saying
   it was not deleted.
6. **Given** a fading row, **When** a screen reader user is on the page, **Then** they hear that
   the entry will be deleted with Undo available (for example "Lunch on Oct 14 removed. Undo
   available for 5 seconds."), and hear again when it is deleted or restored.

---

### Edge Cases

- **Several removes in a row**: the user can remove several rows one after another; each fades
  and has its own Undo and its own 5 seconds, counted from its own remove click.
- **Fading row and other actions**: while a row is fading, its own date, edit and remove are
  unavailable; other rows stay fully usable, and the user can still open a new row with + or
  from another row's date.
- **Opening a row while one fades**: opening a new or edit row does not cancel or speed up a
  pending delete; it still happens when its 5 seconds are up.
- **Leaving the screen**: if the user switches month, year or tab (Spendings / Income), opens
  Settings or starts an import while rows are fading, those deletes are carried out at once
  rather than lost or silently undone, and the totals on the next screen reflect them within 1
  second.
- **Closing or reloading the page, or signing out**: if the page is closed or reloaded, or the
  user signs out (which leaves the page), while a row is fading, the entry is not deleted (it is
  still there next time). Keeping an entry the user meant to delete is the safer mistake.
- **Signed out during the wait**: if the session has expired when the delete is carried out, the
  user is sent to the signed-out page as today, and the entry is not deleted.
- **Entry already gone**: if the entry was deleted elsewhere (another tab or device) before the 5
  seconds end, the row simply leaves the table with no error.
- **Hover-only actions**: the Undo button is never hover-only; it stays visible for the whole 5
  seconds even when the pointer leaves the row.
- **Clicking a date in the new or edit row itself**: the date field of an open row keeps its
  normal calendar behaviour; quick add applies only to dates of saved rows.
- **Dates on another month**: rows only ever show dates of the shown month, so a quick-add entry
  is saved into the month on screen and the view does not switch.

## Requirements *(mandatory)*

### Functional Requirements

**Quick add from a date**

- **FR-001**: The date of every saved row in the desktop Spendings and Income tables MUST be
  clickable when no row is open for adding or editing.
- **FR-002**: Clicking a row's date MUST open the new-entry row at the top of the table, exactly
  as + does, except that the date field is filled in with that row's date, the calendar stays
  closed, and focus goes to the merchant field.
- **FR-003**: A new row opened from a date MUST follow every rule of a new row opened with +
  (validation, autofill, Tab/Enter/Escape behaviour, save and cancel), and its date MUST stay
  editable through the calendar.
- **FR-004**: While any row is open for adding or editing, dates of other rows MUST NOT open a
  new row and MUST NOT look clickable.
- **FR-005**: A clickable date MUST show a pointer cursor and hover style, carry the tooltip and
  accessible name "Add on {date}" in the table's short date format, be reachable with Tab, and
  open the new row with Enter or Space.

**Undo for delete**

- **FR-010**: Clicking remove MUST NOT delete the entry at once. The row MUST stay in place at
  reduced opacity (about 50%) for 5 seconds, its edit and remove buttons replaced by an Undo
  button with an undo arrow icon and the accessible name "Undo", visible without hovering.
- **FR-011**: After clicking remove, focus MUST move to that row's Undo button.
- **FR-012**: Clicking Undo within the 5 seconds MUST restore the row to normal with nothing
  deleted, and return focus to the row's remove button.
- **FR-013**: When 5 seconds pass without Undo, the entry MUST be deleted, the row MUST leave the
  table, and the footer, month line, total card and breakdown cards MUST update as for a delete
  today (within 1 second of the delete being carried out).
- **FR-014**: Totals and breakdowns MUST keep counting a fading entry until it is deleted.
- **FR-015**: If the delete fails, the row MUST return to normal and the user MUST see the
  existing "not deleted" message. If the entry no longer exists, the row MUST leave the table
  with no error. If the session has expired, the existing signed-out handling MUST apply.
- **FR-016**: Each removed row MUST have its own Undo and its own 5 seconds; several rows can be
  fading at once. A fading row's date, edit and remove MUST be unavailable; other rows MUST
  stay usable.
- **FR-017**: Each delete MUST be sent at most once, however often remove or Undo is clicked.
- **FR-018**: Leaving the screen within the app (another month, year or tab, Settings, import)
  MUST carry out all pending deletes at once. Closing or reloading the page, or signing out, MUST
  leave pending entries undeleted.
- **FR-019**: When a row starts fading, and when it is restored or deleted, screen readers MUST
  be told (for example "Lunch on Oct 14 removed. Undo available for 5 seconds.").

**Scope**

- **FR-020**: The phone view MUST NOT change: its delete keeps its confirmation sheet and has no
  quick add from a date.
- **FR-021**: How entries are stored and deleted on the server MUST NOT change; the 5-second wait
  and Undo happen before the delete is requested.

### Key Entities

- **Entry** (a spending or an income entry): unchanged. Date, merchant, category, account,
  amount.
- **Pending delete**: an entry the user has removed whose 5 seconds have not yet ended. It exists
  only on the user's open screen; it is not stored, and the entry is untouched until the wait
  ends.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: Adding a second entry for a day already in the table takes one click to get a new
  row with the right date, against at least three today (+, open calendar, pick day).
- **SC-002**: In 100% of tries, clicking Undo within 5 seconds leaves the entry in place after a
  page reload.
- **SC-003**: In 100% of tries, a removed entry left alone is gone from the table, and from the
  totals, between 5 and 6 seconds after the remove click.
- **SC-004**: Every flow in this spec (quick add by click and by keyboard, undo, delete after the
  wait, several pending deletes, leaving the screen with a pending delete) is covered by an
  automated end-to-end test on both the Spendings and Income tabs.
- **SC-005**: The phone view's behaviour and the desktop's other add, edit and delete behaviour
  pass their existing end-to-end tests unchanged, apart from tests that expected an immediate
  delete.

## Assumptions

- **Both tables**: "spendings table" covers the Income table too, because feature 009 made Income
  follow the same mechanics and UI as Spendings. If the user wants Spendings only, the Income
  half is simply dropped.
- **"Clicked on the calendar" means the table's date**: the description says "the date just
  clicked on the calendar"; this is read as the date shown in the clicked row of the table.
- **Calendar stays closed**: because the user has just chosen the day by clicking it, the new row
  opens with its calendar closed and focus in merchant, unlike + which opens the calendar.
- **Delete delay is client-side**: the entry is only deleted once the 5 seconds end; there is no
  server-side "soft delete" or restore, so nothing changes in the backend or its contract.
- **No toast**: the Undo lives in the faded row itself, as described, not in a separate
  notification bar.
- **Fixed 5 seconds**: the wait is not configurable and does not pause on hover.
- **Desktop only**: the phone already asks for confirmation before deleting, which covers the
  same risk.
