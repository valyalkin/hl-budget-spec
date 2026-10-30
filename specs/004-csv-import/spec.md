# Feature Specification: Import Spendings from CSV

**Feature Branch**: `004-csv-import`

**Created**: 2026-10-29

**Status**: Deployed to the homelab 2026-10-30: backend `10-b9db8ec` (hl-budget-backend#5, migration V4 applied), frontend `12-f4425a5` (hl-budget-frontend#5); Argo CD Synced/Healthy. Live e2e: the full suite passed against the deployed app (import and a11y specs after the test fixes in hl-budget-frontend#6). Quickstart §2–§3 not repeated by hand; the e2e journeys cover the same steps. README Features row: #8.

**Input**: User description: "I would like to develop a new feature for the application which would introduce the "Upload CSV" functionality. The idea is that there is a button on top of overall spending for a month which is "Import CSV". Clicking on it opens a window on the current window where the user can either get the file from the filesystem or drag and drop it there. Sample CSV is also available there. Once imported and processed, another window opens which is the same as the spendings table with the same mechanics, but it should contain the data from the CSV, and whatever is missing can be filled in by the user. User is free to delete and edit any fields, but the "Add Spendings" button is greyed out until all fields are valid. Then this adds spendings (regardless of the date) and goes back to the home screen. User can also Cancel. Designs are available here https://claude.ai/artifact/QTxGKQCY8eE5SUq7387QCZ#page-dcb81aee8d83 and done using claude design."

## Scope

This feature lets a signed-in user add many spendings at once from a CSV file, such as a bank or
card export. The user picks or drops a file, checks and completes every row in a review table
that works like the Spendings table, and adds all rows in one step.

The look, layout and wording follow the design canvas "HL-Budget"
(https://claude.ai/artifact/QTxGKQCY8eE5SUq7387QCZ), artboards "Import CSV window" and "Review
import". Where this spec and the design differ, this spec wins (it adds the Spendings table's
autofill and keyboard entry to the review table, refuses future dates as the rest of the app
does, replaces the design's "Try a sample file" with "Download sample CSV", and reads ambiguous
dates day first instead of month first).

It builds on feature 002 (spendings, categories and accounts on the server) and feature 003
(the Spendings screen, its entry row, autofill and keyboard rules). Nothing is saved until the
user confirms, and then all rows are saved together or none are.

## Clarifications

### Session 2026-10-29

- Q: How should the sample CSV be offered? → A: As a download only. The Import CSV window has a
  "Download sample CSV" link instead of the design's "Try a sample file" (User Story 3, FR-005).
- Q: How are ambiguous dates like 03/04/2026 read? → A: Always day first (DD/MM/YYYY), matching
  Canadian and Amex exports; month-first dates are not supported (Edge Cases, FR-006).
- Q: Should rows matching existing spendings be flagged or removed? → A: Neither. There is no
  duplicate check; every row is imported as shown (Edge Cases, FR-007).

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Import a clean CSV file (Priority: P1)

On the Spendings screen the user clicks "Import CSV", the button above the month's total card.
An "Import CSV" window opens over the screen. The user drops a CSV export onto the drop area (or
clicks "Choose file" and picks it). The file is read and a "Review import" window shows every
row in a table with date, merchant, category, account and amount. All rows look good, so the
user clicks "Add N spendings". The spendings are added, whatever their dates, the windows close,
and the user is back on the Spendings screen with a short message confirming how many were
added.

**Why this priority**: This is the whole point of the feature: getting a batch of spendings in
without typing them one by one.

**Independent Test**: Prepare a CSV with a header row and 5 complete rows spread over September
and October, all dated today or earlier and using existing categories and accounts. Import it,
check the review shows 5 rows with "All rows look good", click "Add 5 spendings", and check all
5 appear in the right months on the Spendings screen and on the server with the exact values
from the file.

**Acceptance Scenarios**:

1. **Given** the Spendings screen, **When** the user looks above the month total card, **Then**
   they see an "Import CSV" button.
2. **Given** the user clicks "Import CSV", **When** the window opens, **Then** it shows the title
   "Import CSV", a short explanation ("Upload a CSV export from your bank or card. You'll check
   every row before anything is added."), a drop area reading "Drop your CSV file here" with a
   "Choose file" button, a note on what the file should contain, a "Download sample CSV" link
   and a Cancel button. The screen behind stays visible but cannot be used.
3. **Given** the Import CSV window, **When** the user drags a file over the drop area, **Then**
   the drop area is highlighted; **when** they drop a CSV file there, **Then** it is read.
4. **Given** the Import CSV window, **When** the user clicks "Choose file" and picks a CSV file,
   **Then** it is read the same way as a dropped file.
5. **Given** a file that was read, **When** reading finishes, **Then** the "Review import"
   window replaces the Import CSV window, showing the file name and row count ("transactions.csv
   · 10 rows") and one table row per data row of the file, in file order.
6. **Given** every row is complete and valid, **When** the review is shown, **Then** it shows
   "All rows look good", a footer with the number of spendings and their total ("10 spendings ·
   total $358.03"), and an enabled "Add 10 spendings" button.
7. **Given** a review with all rows valid, **When** the user clicks "Add N spendings", **Then**
   all rows are added as spendings regardless of which month or year they fall in, the window
   closes, and the Spendings screen shows the month of the latest imported date with the
   message "Added N spendings from [file name]". The month line, table, footer and cards all
   include the new spendings.
8. **Given** a category or account name in the file that matches an existing one in any letter
   case, **When** the rows are shown, **Then** the existing name is used (for example "visa"
   becomes "Visa").

---

### User Story 2 - Fix and complete rows before adding (Priority: P1)

Real exports are rarely perfect: a row may have no category, no account, an unreadable date or
a missing amount. The review table highlights every field that needs attention and says how
many rows need work. The user edits any field directly in the table, with the same calendar,
autofill and keyboard entry as the Spendings table, and removes rows they don't want. "Add
spendings" stays greyed out until every remaining row is valid.

**Why this priority**: Without fixing rows in place, any imperfect file would be unusable; the
user explicitly asked that missing data can be filled in and that adding is blocked until
everything is valid.

**Independent Test**: Import a file in which one row has no category, one has no account, one
has no date and one has no amount. Check the review says "4 rows need attention", those four
fields are highlighted, and "Add spendings" is greyed out. Fill each field in, check the
highlight disappears field by field and the button becomes "Add N spendings" once the last one
is fixed. Add and check the server has the values as edited.

**Acceptance Scenarios**:

1. **Given** rows with missing or invalid fields, **When** the review is shown, **Then** each
   such field is highlighted and marked as invalid for assistive technology, an empty field
   shows the placeholder "Required" (an empty date shows "Pick a date"), and a badge reads
   "N rows need attention" (or "1 row needs attention").
2. **Given** at least one row needs attention, **When** the user looks at the footer, **Then**
   "Add spendings" is shown greyed out and cannot be activated by mouse or keyboard.
3. **Given** a highlighted field, **When** the user enters a valid value, **Then** its highlight
   disappears at once, and the badge, the footer total and the button update without any
   further action.
4. **Given** any field of any row, valid or not, **When** the user edits it, **Then** the
   change is kept in the review table; nothing is saved until "Add N spendings" is clicked.
5. **Given** the date field of a row, **When** the user opens it, **Then** the same calendar as
   in the Spendings table opens (month arrows, "Today" shortcut, days after today cannot be
   picked) at the row's date, or at the current month when the date is missing.
6. **Given** the merchant, category or account field of a row, **When** the user types, **Then**
   the same autofill and suggestion list as in the Spendings table appear, ranked by how often
   the user has used each value; accepting a past merchant fills in that row's category and
   account only when they are still empty.
7. **Given** a field in the review table, **When** the user presses Tab, Shift+Tab, Enter or
   the Left/Right arrows, **Then** focus moves as in the Spendings table's entry row (date,
   merchant, category, account, amount); Tab from a row's amount moves to the next row's date.
8. **Given** a row, **When** the user clicks its remove button, **Then** the row is taken out
   of the review at once (no confirmation), and the row count, badge, total and button update.
9. **Given** the user removes every row, **When** the table is empty, **Then** it shows "No rows
   left to import." and "Add spendings" is greyed out.
10. **Given** rows whose category or account name does not exist yet, **When** the user adds
    them, **Then** those categories and accounts are created together with the spendings, as in
    the Spendings table; a name typed in several rows in different letter cases creates one item.

---

### User Story 3 - Download the sample file (Priority: P2)

A user who isn't sure what the file should look like downloads the sample CSV from the Import
CSV window, looks at it, and shapes their own file the same way (or imports the sample itself).

**Why this priority**: Helpful for learning the format, but not needed to import real data.

**Independent Test**: Open Import CSV, click "Download sample CSV", check a file named
`sample-spendings.csv` is saved with a header row and 10 data rows, then import that file and
check the review shows the same 10 rows.

**Acceptance Scenarios**:

1. **Given** the Import CSV window, **When** the user clicks "Download sample CSV", **Then** the
   browser saves the sample file; the window stays open and nothing is imported.
2. **Given** the sample file, **When** the user opens it, **Then** it has the header row
   `Date,Merchant,Category,Account,Amount`, dates written as DD/MM/YYYY and `YYYY-MM-DD`, and
   complete rows as well as rows with an empty category, account or amount, so it shows that
   missing values can be filled in later.
3. **Given** the downloaded sample, **When** the user imports it like any other file, **Then**
   the review shows its rows, with the empty fields highlighted.

---

### User Story 4 - Cancel an import (Priority: P2)

At any point before adding, the user can back out and nothing is saved.

**Why this priority**: Necessary for safety, but simple.

**Independent Test**: Import a file, edit some rows, click Cancel, and check no spending,
category or account was created.

**Acceptance Scenarios**:

1. **Given** the Import CSV window, **When** the user clicks Cancel, the close button, or presses
   Escape, **Then** the window closes and the Spendings screen is shown unchanged.
2. **Given** the Review import window, **When** the user clicks Cancel or the close button (or
   presses Escape while no calendar or suggestion list is open), **Then** the rows and any edits
   are discarded, nothing is saved, and the user returns to the Import CSV window to pick
   another file or cancel from there.
3. **Given** the user cancelled a review, **When** they open the same file again, **Then** the
   review starts from the file's contents; earlier edits are not remembered.

---

### Edge Cases

- **Not a CSV file**: a file that is not a CSV (by name or type) is refused in the Import CSV
  window with "That doesn't look like a CSV file. Choose a .csv export." and no review opens.
- **Unreadable file**: if the file cannot be read, the window shows "The file couldn't be read.
  Try again."
- **No data rows**: an empty file, or a file with only a header row, shows "No spendings found
  in that file. Check that it is a CSV with a header row and at least one data row."
- **Too large**: a file over 1 MB or with more than 1,000 data rows is refused with a message
  stating the limit; no review opens.
- **Several files dropped**: only the first file is read.
- **Column order and names**: columns are found by their header names in any order and letter
  case: date (also "posted", "day"), merchant (also "description", "payee", "vendor", "name",
  "memo", "store", "details"), category, account (also "card", "source") and amount (also
  "total", "value", "debit", "price", "cost"). Columns that match none of these are ignored.
- **Missing columns**: a column the file does not have (for example no category column) leaves
  that field empty in every row, to be filled in on the review.
- **No header row**: when fewer than two columns can be recognised from the first row, the file
  is read as having no header, with columns in the order date, merchant, category, account,
  amount, and the first row is treated as data.
- **Quoted values**: values in double quotes may contain commas, quotes (written as two double
  quotes) and line breaks; blank lines are skipped; a leading byte-order mark is ignored.
- **Dates**: two forms are recognised: year first, `YYYY-MM-DD` (also with `/` or `.`), and day
  first, `DD/MM/YYYY` (also with `-` or `.`, one- or two-digit day and month). A day-first date
  is always read as day then month, so 03/04/2026 is 3 April 2026; month-first (US) dates are
  not supported, and one like 10/27/2026 is impossible as day/month, so it is left empty.
  A time after the date (`2026-10-28 14:03`) is ignored. Impossible dates (such as 2026-02-30)
  and anything else leave the date empty, to be picked on the review.
- **Future dates**: a date after today is highlighted as needing attention ("Pick a date that is
  today or earlier."), consistent with the rest of the app.
- **Amounts**: currency symbols, spaces and thousands separators are ignored; a negative amount
  (`-18.90` or `(18.90)`) is taken as a positive spending of 18.90; amounts are rounded to 2
  decimal places. An empty, zero or non-numeric amount is left empty and highlighted.
- **Long values**: a merchant over 255 characters, or a category or account over 50, is
  highlighted as needing attention, with the same limits as entering a spending by hand.
- **Spaces**: leading and trailing spaces are trimmed and runs of spaces inside names collapsed.
- **Duplicates**: rows are not checked against spendings already recorded, nor against each
  other. Importing the same file twice adds its spendings twice; the user removes repeats in the
  review or deletes them afterwards.
- **Adding fails**: if the server refuses or cannot be reached when adding, nothing is added (no
  spendings, categories or accounts), the review stays open with every row and edit as it was,
  and a short message explains the problem. A row the server finds invalid is highlighted. If
  the server could not be reached but did in fact save the rows, trying again adds nothing:
  the review closes with "These spendings were already added." and the Spendings screen shows
  them.
- **Double add**: clicking "Add N spendings" twice, or pressing Enter twice, adds the rows once;
  the button is disabled while adding.
- **Session expiry**: if the session ends during the review, the next action shows the signed-out
  screen; the rows are lost and nothing is added.
- **Data changed elsewhere**: if a category or account named in the rows is removed in another
  tab before adding, it is simply created again when the rows are added.
- **Open entry row**: while a row is open for adding or editing in the Spendings table, the
  "Import CSV" button is unavailable, like the + button; the user saves or cancels the row first.
- **Leaving the page**: closing or reloading the browser tab during a review discards the rows;
  nothing is added.

## Requirements *(mandatory)*

### Functional Requirements

**Opening the import**

- **FR-001**: The Spendings screen MUST show an "Import CSV" button above the month total card,
  with an upload icon, matching the design.
- **FR-002**: The button MUST open the Import CSV window as a modal over the current screen; the
  screen behind MUST NOT be usable while it is open, and keyboard focus MUST stay inside it.
- **FR-003**: The Import CSV window MUST show the explanation, a drop area that accepts a
  dragged file and highlights while a file is dragged over it, a "Choose file" button that opens
  the system file picker limited to CSV files, a note describing the expected columns, the
  sample (FR-005), and Cancel and close buttons.
- **FR-004**: The windows' look, layout and wording MUST match the design artboards "Import CSV
  window" and "Review import", except where this spec adds or changes behaviour.
- **FR-005**: The Import CSV window MUST offer a "Download sample CSV" link that saves a sample
  file (`sample-spendings.csv`) without closing the window or importing anything. The sample
  MUST use the column names `Date,Merchant,Category,Account,Amount` and include complete rows
  and rows with missing values.
- **FR-006**: Dates in the file MUST be read as year-month-day (`YYYY-MM-DD`) or day-month-year
  (`DD/MM/YYYY`); a date with the year last is always read day first.
- **FR-007**: Rows MUST NOT be checked for duplicates, either against existing spendings or
  within the file.

**Reading the file**

- **FR-010**: The system MUST read one CSV file of up to 1 MB and 1,000 data rows, and refuse
  larger files with a message stating the limit.
- **FR-011**: The system MUST find the date, merchant, category, account and amount columns by
  their header names, in any order and letter case, using the alternative names listed under
  Edge Cases; when fewer than two columns can be recognised it MUST read the file as having no
  header, in the fixed order date, merchant, category, account, amount.
- **FR-012**: The system MUST support standard CSV quoting (commas, double quotes and line
  breaks inside quoted values), skip blank lines and ignore a leading byte-order mark.
- **FR-013**: The system MUST turn each data row into one review row, keeping file order, and
  normalise values as described under Edge Cases (dates, amounts, spaces).
- **FR-014**: A category or account value that matches one of the user's existing names in any
  letter case MUST be shown with the existing name.
- **FR-015**: A value that cannot be read (date, amount) or that is absent MUST leave the field
  empty, not refuse the file; only files that are not CSV, cannot be read, are too large or have
  no data rows are refused, each with its message from Edge Cases.

**Reviewing rows**

- **FR-020**: The Review import window MUST show the file name and row count, a status badge
  ("N rows need attention" or "All rows look good"), the hint "Highlighted fields need your
  attention. Remove a row to leave it out.", a table with date, merchant, category, account,
  amount and a remove button per row, and a footer with the number of rows, their total of
  valid amounts, Cancel and the add button.
- **FR-021**: A row field MUST be treated as valid by the same rules as entering a spending by
  hand: a date today or earlier; a merchant, category and account that are not empty and within
  their length limits; an amount greater than zero with at most 2 decimal places.
- **FR-022**: Every invalid field MUST be highlighted and marked invalid for assistive
  technology; validity, the badge, the footer and the add button MUST update as the user types.
- **FR-023**: Every field of every row MUST be editable directly in the table, using the same
  calendar, autofill, suggestion list and keyboard navigation as the Spendings table's entry
  row (feature 003, FR-030 to FR-058), except that a row is never saved on its own.
- **FR-024**: Each row's remove button MUST take the row out of the review immediately, without
  confirmation; when no rows remain, the table MUST show "No rows left to import."
- **FR-025**: The table MUST show all rows (no paging) and scroll within the window when they do
  not fit; calendars and suggestion lists MUST always show in full.

**Adding**

- **FR-031**: The add button MUST be greyed out and unusable, labelled "Add spendings", while any
  row is invalid or no rows remain; otherwise it MUST be enabled and labelled "Add N spendings"
  (or "Add 1 spending").
- **FR-032**: Adding MUST save every remaining row as a spending with its date, whatever month or
  year that is, and create any category or account names that don't exist yet (one per name,
  ignoring letter case).
- **FR-033**: Adding MUST be all or nothing: if any row is refused or the save fails, no
  spending, category or account is created, and the review stays open with all rows and edits.
- **FR-034**: Adding MUST happen at most once per user action, even if the button is clicked or
  Enter pressed repeatedly.
- **FR-035**: After a successful add, both windows MUST close and the Spendings screen MUST show
  the year and month of the latest imported date, with the message "Added N spendings from [file
  name]"; the month line, table, footer, total card and By category card MUST include the new
  spendings.

**Cancelling**

- **FR-040**: Cancel, the close button and Escape in the Import CSV window MUST close it with no
  changes.
- **FR-041**: Cancel, the close button and Escape (when no calendar or suggestion list is open)
  in the Review import window MUST discard the rows and edits, save nothing, and return to the
  Import CSV window.

**General**

- **FR-050**: Only a signed-in user MAY import; imported spendings, categories and accounts MUST
  belong to that user only.
- **FR-051**: The file's contents MUST NOT be stored anywhere except as the spendings the user
  adds, and MUST NOT be written to logs.
- **FR-052**: Both windows MUST be usable with the keyboard alone and meet the same
  accessibility bar as the rest of the app (labelled controls, visible focus, announced errors).

### Key Entities *(include if feature involves data)*

- **Import file**: the CSV the user picked or dropped; its name and rows. Held only while the
  windows are open; never stored.
- **Import row**: one data row of the file as the user is reviewing it: date, merchant,
  category, account, amount, and whether each is valid. Exists only in the review.
- **Spending, Category, Account**: as in feature 002. Adding creates spendings, and categories
  and accounts whose names don't exist yet.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: A user can import a clean 30-row file, from clicking "Import CSV" to seeing the
  spendings on the Spendings screen, in under 1 minute.
- **SC-002**: A user can import a 30-row file in which 5 rows need one field each filled in, in
  under 3 minutes, without leaving the review.
- **SC-003**: For a well-formed file with recognisable headers, 100% of rows arrive in the review
  with their date, merchant, category, account and amount exactly as in the file.
- **SC-004**: In 100% of failed or cancelled imports, no spending, category or account is
  created; in 100% of successful imports, the number of spendings added equals the number of
  rows shown when the user clicked add.
- **SC-005**: A 1,000-row file opens in the review in under 3 seconds, and editing a field there
  updates its highlight and the totals with no noticeable delay.
- **SC-006**: Every invalid field is highlighted before the user tries to add; the add button is
  never enabled while any row is invalid.

## Assumptions

- **Design is the source for look and wording**: colours, type, spacing and messages follow the
  design artboards; this spec adds the Spendings table's autofill and keyboard entry, and the
  future-date rule, which the design does not show.
- **Cancel in the review returns to the Import CSV window**, as in the design, so the user can
  pick another file; cancelling there returns to the Spendings screen.
- **Negative amounts are spendings**: bank exports often show spendings as negative numbers, so
  the sign is dropped. Refunds and credits in a file are imported as spendings unless the user
  removes those rows.
- **Comma-separated, UTF-8 files only**: semicolon- or tab-separated files and other text
  encodings are out of scope; such files will usually show as needing attention or as having no
  data.
- **Single currency**: amounts are taken in the app's one currency; a currency column, if
  present, is ignored.
- **Landing month**: after adding, the Spendings screen shows the month of the latest imported
  date (as in the design), which may differ from the month shown before importing.
- **No column mapping step**: columns are recognised automatically; there is no screen to map
  columns by hand. Anything not recognised is filled in on the review.
- **No saved import history**: the app does not remember which files were imported.
- **Desktop first**: like feature 003, the windows are designed for desktop and laptop browsers;
  on narrow screens the review table scrolls sideways.
- **Out of scope**: other file formats (Excel, OFX, QIF, PDF), importing income, splitting a
  row into several spendings, and undoing an import after it has been added (spendings can be
  deleted one by one as usual).

## Dependencies

- **Feature 002**: spendings, categories and accounts on the server, the validation rules and
  length limits, and creating a missing category or account together with a spending.
- **Feature 003**: the Spendings screen, the entry row's calendar, autofill, suggestion lists,
  keyboard navigation and messages, which the review table reuses.
- **Server addition for adding many at once**: feature 002 creates one spending per request.
  Adding all rows as one all-or-nothing step (FR-033) and within SC-001 most likely needs a
  backward-compatible server operation that creates many spendings, and their missing
  categories and accounts, in a single step. The plan decides its shape.
