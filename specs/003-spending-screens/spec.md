# Feature Specification: Spending Screens (Web App)

**Feature Branch**: `003-spending-screens`

**Created**: 2026-10-28

**Status**: Implemented 2026-10-28 on branch `003-spending-screens` in hl-budget-backend and hl-budget-frontend (not yet merged or deployed). Gates green, including the signed-in e2e journeys and axe scans; by-hand quickstart checks pending. Revised the same day after the user's hands-on review (Clarifications, post-implementation review; tasks.md Phase 11)

**Input**: User description: "Now I would like to build frontend for the budget app functionality - I have designed the pages in claude design - https://claude.ai/code/artifact/be0595bc-5e5a-41cf-9fe0-6264edaa7efa. [Design summary: header with Sign in / Sign out and a user icon; signed-out message; Spendings screen with a year and month line, a compact table of the month's spendings with an inline add row (date calendar, merchant, category, account, amount), a month total card and a By category card with Table / Pie toggle; Settings window with Categories and Accounts sections and removal rules.] Also include deleting functionality for the row. When the user is adding spendings, for categories or account, when the user starts typing, we need to autofill the field with frequently used data which corresponds to whatever the user is typing. When user completes the field, when he presses TAB or ENTER, it needs to move to the next field or save it when all fields are complete. There must be edit and remove buttons on hover as well."

## Scope

This feature delivers the screens of HL Budget's first business capability: a signed-in user
sees their spendings month by month, records, edits and deletes spendings directly in the
table, sees where the month's money went by category, and manages their categories and
accounts in a Settings window.

The look and layout follow the design canvas "HL-Budget"
(https://claude.ai/code/artifact/be0595bc-5e5a-41cf-9fe0-6264edaa7efa), artboards
"Spendings", "Add in the table" and "Settings window". Where this spec and the design differ,
this spec wins (it adds editing, deleting and keyboard-driven entry, which the design does not
show, and the pie chart shows every category with spending instead of the design's top six plus
"Everything else").

Data, rules and sign-in come from features 001 (sign-in) and 002 (spendings, accounts and
categories on the server). This feature adds screens, and these server changes:

- **Two rules**: an account cannot be removed when it is the user's last one (FR-086), and
  saving a spending with a category or account name that doesn't exist yet creates it in the
  same step, so nothing is created if the spending is refused (FR-035, Edge Cases).
- **Three read-only lookups**: month totals for a year, totals per category for a month, and
  past merchants for autofill (see Dependencies).

All server changes must be backward compatible.

## Clarifications

### Session 2026-10-28

- Q: Should the "you can't remove your last account" rule be enforced by the backend, or only by
  the screen? → A: The backend enforces it: removing a user's only account is refused as a
  business-rule failure, and the screen shows "You need at least one account."
- Q: Should the merchant field autofill too, and fill in category and account? → A: Yes. The
  merchant autofills from the user's past merchants (most used first); accepting a past merchant
  fills in the category and account of that merchant's most recent spending, when those fields
  are still empty. Both stay editable.
- Q: When you click remove on a spending row, should it confirm first or offer undo? → A: Neither.
  Remove deletes the spending at once, with no confirmation and no undo.
- Q: Which categories does the By category overview show? → A: Only categories with spending in
  the selected month, all of them, in both Table and Pie. The pie has one slice per such
  category; there is no limit of six and no "Everything else" slice (a change from the design).

Post-implementation review (the user tried the screens; each answer is the user's request):

- Q: The calendar was cut off inside the table; should popups be clipped by the table? → A: No.
  The calendar and the merchant, category and account suggestion lists MUST always show in full
  (FR-034, FR-051).
- Q: The merchant field was too narrow to read its placeholder; how wide must it be? → A: Wide
  enough to read. The merchant column keeps a usable minimum width, and the summary cards move
  below the table on narrower screens (FR-024).
- Q: Should a new row go straight to the calendar? → A: Yes. A new row opens its calendar by
  itself; picking a day moves on to the merchant (FR-030).
- Q: Should arrow keys move back and forth along the row? → A: Yes, Left and Right (FR-053).
- Q: Which date should a new spending start with when another month or year is shown? → A: Not
  today: the shown month (today in the current month, otherwise its 1st) (FR-030).
- Q: Can Enter on the page start a new spending? → A: Yes, when nothing has focus (FR-030).

## User Scenarios & Testing *(mandatory)*

### User Story 1 - See my spendings for a month (Priority: P1)

A signed-in user opens the app and lands on the Spendings screen for the current month. A line
across the top shows the year with arrows to change it, and the twelve months of that year,
each with its total. The user clicks a month to see its spendings in a compact table (date,
merchant, category, account, amount), newest first, with a footer giving the number of
spendings and their total. A card shows the month's total and spending count.

**Why this priority**: Viewing spendings is half the purpose of the app and the screen every
other story lives on.

**Independent Test**: With spendings already recorded through the server in September and
October of the current year, open the app, check October is selected and its table, footer and
card match the recorded data, then click September and check the same for September.

**Acceptance Scenarios**:

1. **Given** a signed-in user with spendings this month, **When** they open the app, **Then**
   the current month of the current year is selected and its spendings are listed newest date
   first, with a footer showing the count (for example "14 spendings") and the total.
2. **Given** the month line, **When** the user looks at it, **Then** each past and current month
   shows its total (or a dash when it has none), and months after the current month are greyed
   out and cannot be selected.
3. **Given** the month line, **When** the user clicks the previous-year arrow, **Then** the line
   shows the previous year's months and totals. The next-year arrow is unavailable while the
   current year is shown.
4. **Given** a month with no spendings, **When** the user selects it, **Then** the table shows
   "No spendings recorded in [Month Year]." instead of rows, and the card shows a total of zero
   and 0 spendings.
5. **Given** a signed-out visitor, **When** they open the app, **Then** they see "You're signed
   out" with a short explanation and a Sign in button instead of any spendings.

---

### User Story 2 - Record a spending in the table (Priority: P1)

The user clicks the + button in the table header. A new editable row appears at the top of the
table with fields for date, merchant, category, account and amount. The date defaults to today
when the current month is shown, otherwise to the 1st of the shown month, and the calendar opens
by itself. The user types the merchant, picks or types a category and account, types
the amount and saves with the check mark (or the keyboard). The new spending appears in the
table, and the view switches to the month of its date.

**Why this priority**: Recording spendings is the core action of the app.

**Independent Test**: With one account, click +, fill every field, save, and check the spending
appears in the table and in the server's data with the exact values entered.

**Acceptance Scenarios**:

1. **Given** the Spendings screen, **When** the user clicks +, **Then** an editable row opens at
   the top of the table with its calendar open, the date set to today when the current month is
   shown or to the 1st of the shown month otherwise; picking a day moves focus to the merchant.
2. **Given** the date field, **When** the user opens it, **Then** a calendar shows the month of
   the current date value, with previous and next month arrows and a "Today" shortcut; days
   after today cannot be picked.
3. **Given** a complete row, **When** the user clicks the check mark, **Then** the spending is
   saved, the row closes, the view switches to the year and month of the spending's date, and
   the month line, table, footer, total card and By category card all include it.
4. **Given** an open row, **When** the user clicks X (or presses Escape), **Then** the row closes
   and nothing is saved.
5. **Given** a row with a missing or invalid field, **When** the user tries to save, **Then**
   nothing is saved, the row stays open with what was typed, and a message names the first
   problem: "Pick a date.", "Add a merchant name.", "Choose or type a category.", "Choose or type
   an account." or "Enter an amount greater than zero."
6. **Given** a category or account name typed in the row that does not exist yet (ignoring
   letter case), **When** the user saves, **Then** that category or account is created and the
   spending is saved with it. A name that matches an existing item in any letter case uses the
   existing item.
7. **Given** a save that the server refuses or that fails to reach the server, **When** it
   happens, **Then** the row stays open with what was typed and the user sees a short message
   explaining what went wrong; nothing is shown as saved.

---

### User Story 3 - Fast keyboard entry with autofill (Priority: P1)

While entering a spending, the user can do everything from the keyboard. In the merchant,
category and account fields, as soon as the user starts typing, the field autofills with the
user's most frequently used matching value, and a dropdown lists the other matches. Accepting a
past merchant also fills in the category and account that merchant was last recorded with. Tab or Enter accepts the
field and moves on; when every field is complete, Enter saves the spending.

**Why this priority**: The user enters many spendings, often in batches from statements. The
user explicitly asked for autofill and Tab/Enter navigation, and it is what makes entry fast.

**Independent Test**: With "Groceries" used 10 times and "Gifts" used once, open a new row, type
a merchant, press Tab, type "g" in category and check "Groceries" is filled in; press Tab, type
the account's first letter, press Tab, type an amount, press Enter, and check the spending is
saved without touching the mouse.

**Acceptance Scenarios**:

1. **Given** the category field, **When** the user types "g", **Then** the field fills in the
   most frequently used category whose name starts with "g" (for example "Groceries"), with the
   untyped remainder shown as a selected suggestion, and the dropdown lists all matching
   categories, most used first.
2. **Given** an autofilled suggestion, **When** the user keeps typing, **Then** the suggestion
   updates to the most used item matching everything typed so far; when nothing matches, no
   suggestion is shown and what the user typed is kept as a new name.
3. **Given** an autofilled suggestion, **When** the user presses Tab or Enter, **Then** the
   suggestion is accepted and focus moves to the next field.
4. **Given** an autofilled suggestion, **When** the user presses Backspace, **Then** the
   suggested remainder is removed and only what they typed is kept.
5. **Given** any field in the row, **When** the user presses Tab, **Then** focus moves to the
   next field in the order date, merchant, category, account, amount; Shift+Tab moves back.
6. **Given** any field in the row, **When** the user presses Enter and every field is complete,
   **Then** the spending is saved. **When** a field is still empty, **Then** focus moves to the
   next field instead.
7. **Given** the amount field (the last field), **When** the user presses Tab or Enter and every
   field is complete, **Then** the spending is saved; **when** a field is still empty, **Then**
   the validation message is shown and focus moves to the first empty field.
8. **Given** the category or account dropdown is open, **When** the user presses the up or down
   arrow keys, **Then** the highlighted option changes, and Enter or Tab picks it.
9. **Given** "most frequently used", **When** suggestions are ranked, **Then** items are ordered
   by how many of the user's spendings use them, highest first, and ties are ordered
   alphabetically.
10. **Given** the user has recorded "Neighbourhood Market" 12 times, most recently in
    "Groceries" on "Visa", **When** they type "nei" in the merchant field of a new row, **Then**
    the field autofills "Neighbourhood Market"; **when** they press Tab, **Then** category is
    filled with "Groceries", account with "Visa", and focus moves to the category field with
    its value selected so it can be overwritten or accepted with Tab.
11. **Given** the category or account field already holds a value (typed by the user, or when
    editing a spending), **When** a past merchant is accepted, **Then** that value is not
    replaced.
12. **Given** a merchant the user has never recorded, **When** they type it, **Then** no
    suggestion is shown, what they typed is kept, and category and account are not filled in.

---

### User Story 4 - Edit and delete a spending from the table (Priority: P1)

When the user hovers over a spending row (or moves keyboard focus into it), edit and remove
buttons appear on the row. Edit turns the row into the same editable row used for adding, filled
with the spending's values. Remove deletes the spending at once.

**Why this priority**: Mistakes happen when entering many spendings; without edit and delete the
user cannot fix them. The user explicitly asked for both.

**Independent Test**: Record a spending, hover its row, edit its amount and category, save, and
check the table and server show the change. Then hover it, click remove, and check it is gone
from the table, the totals and the server.

**Acceptance Scenarios**:

1. **Given** a spending row, **When** the user hovers it or focuses it with the keyboard,
   **Then** edit and remove buttons appear on that row; they are hidden on other rows.
2. **Given** a spending row, **When** the user clicks edit, **Then** the row becomes editable
   with the spending's current date, merchant, category, account and amount, and the same
   calendar, autofill, keyboard and validation rules as adding apply.
3. **Given** an edited row, **When** the user saves, **Then** the spending is updated, and if
   its date moved to another month the view switches to that month; totals everywhere reflect
   the change.
4. **Given** an edited row, **When** the user cancels (X or Escape), **Then** the row shows the
   original values and nothing is changed.
5. **Given** a spending row, **When** the user clicks remove, **Then** the spending is deleted
   at once, with no confirmation and no undo, and the table, footer, month line and cards
   update.
6. **Given** a remove that the server refuses or cannot complete, **When** it happens, **Then**
   the row stays in the table and the user sees a short message saying it was not deleted.
7. **Given** a row is already open for adding or editing, **When** the user starts another add
   or edit, **Then** only one row is editable at a time: the open row must be saved or
   cancelled first (the + and edit buttons are unavailable meanwhile).

---

### User Story 5 - See where the month's money went (Priority: P2)

The By category card shows the selected month's spending by category, as a table or a pie
chart, chosen with a Table / Pie toggle.

**Why this priority**: The breakdown turns a list into insight, but the user can track spending
without it.

**Independent Test**: With October spendings in eight categories, select October, check the
table lists those eight categories (and none of the unused ones) with amount, share and a total
row, then switch to Pie and check eight slices with a legend.

**Acceptance Scenarios**:

1. **Given** the By category card in Table view, **When** a month with spendings is selected,
   **Then** each category used that month is listed with its amount and its share of the month's
   total, largest first, followed by a total row.
2. **Given** the card in Pie view, **When** a month with spendings in eight categories is
   selected, **Then** a donut chart shows eight slices, one per category, largest first, with a
   legend naming each slice with its amount and share. There is no "Everything else" slice.
3. **Given** categories with no spending in the selected month, **When** either view is shown,
   **Then** those categories do not appear.
4. **Given** a month with no spendings, **When** the card is shown, **Then** it shows a short
   empty message instead of a table or chart.
5. **Given** the user switches between Table and Pie, **When** they then change month, **Then**
   the chosen view is kept.

---

### User Story 6 - Manage categories and accounts in Settings (Priority: P2)

The user icon in the header opens a Settings window with sections listed on the left:
Categories and Accounts. In Categories the user sees the 17 default categories and their own,
and can add and remove their own. In Accounts the user sees their accounts with how many
spendings use each, and can add and remove accounts.

**Why this priority**: Categories and accounts can also be created while recording spendings
(Story 2), so Settings is mainly for tidying up.

**Independent Test**: Open Settings, add a category "Coffee" and an account "Cash", check they
appear in the Settings lists and in the entry row dropdowns, then remove both.

**Acceptance Scenarios**:

1. **Given** the Settings window, **When** the user opens Categories, **Then** they see the 17
   default categories (marked as defaults, with no remove button) and their own categories, each
   with how many spendings use it.
2. **Given** Categories, **When** the user adds a name that is empty, **Then** they see "Type a
   name for the category."; **when** the name matches an existing category in any letter case,
   **Then** they see "“[Name]” already exists." and nothing is added.
3. **Given** a custom category no spending uses, **When** the user removes it, **Then** it
   disappears from Settings and from the entry dropdowns.
4. **Given** a custom category that spendings use, **When** the user tries to remove it, **Then**
   nothing is removed and they see "[Name] is used by N spendings, so it can't be removed."
5. **Given** Accounts, **When** the user adds or removes accounts, **Then** the same empty and
   duplicate rules apply, and each account shows its spending count ("3 spendings recorded" or
   "No spendings yet").
6. **Given** an account that spendings use, **When** the user tries to remove it, **Then**
   nothing is removed and they see "[Name] has N spendings, so it can't be removed."
7. **Given** the user has exactly one account, **When** they try to remove it, **Then** nothing
   is removed and they see "You need at least one account."
8. **Given** the Settings window, **When** the user closes it (close button, Escape, or clicking
   outside), **Then** they return to the Spendings screen, which reflects any changes made.

---

### User Story 7 - Sign in and out (Priority: P3)

The header shows Sign in while signed out, and Sign out plus the user icon while signed in.

**Why this priority**: Sign-in already exists from feature 001; this story only places it in the
new header.

**Independent Test**: Sign in from the signed-out screen, check the Spendings screen appears;
sign out, check the signed-out message returns and no spending data is visible.

**Acceptance Scenarios**:

1. **Given** a signed-out visitor, **When** they click Sign in, **Then** they go through the
   existing sign-in and land on the Spendings screen for the current month.
2. **Given** a signed-in user, **When** they click Sign out, **Then** they are signed out and see
   the signed-out message; going back in the browser does not show their spendings.

---

### Edge Cases

- **A user with no accounts**: the entry row still opens; typing an account name creates the
  first account on save. Settings shows an empty Accounts list with the add field.
- **No partly created items on a failed save**: a new category or account typed in the row is
  created together with the spending, as one step. If the spending is refused, nothing is
  created, and the row stays open so the user can fix and retry.
- **Double save**: pressing Enter or clicking the check mark twice quickly saves the spending
  once; the save control is disabled while saving.
- **Future dates**: days after today cannot be picked in the calendar, and a future date that reaches
  the server anyway is refused with "Pick a date that is today or earlier."
- **Amounts**: the amount field accepts digits and one decimal separator with at most 2 decimal
  places; values like "0", "-5" or "abc" are refused with "Enter an amount greater than zero."
  Amounts are shown with the app's single currency and two decimals.
- **Long names**: long merchant, category and account names are shortened with an ellipsis in
  the table and shown in full on hover; names follow feature 002's limits (merchant 255,
  category and account 50 characters).
- **Autofill and an exact match**: when what the user typed exactly matches an item (ignoring
  case), that item is used; the autofill never replaces a complete exact match with a longer
  name.
- **Autofill on edit**: when editing an existing spending, the fields start with the current
  values selected; autofill applies only once the user types, and changing the merchant never
  replaces the category or account already set.
- **Merchant spellings**: "loblaws" and "Loblaws " count as the same past merchant; the
  spelling from the most recent spending is suggested. The merchant is still free text, so the
  user can save a new spelling.
- **Removing the last spending in a month**: the table switches to the empty message and the
  month's total becomes a dash.
- **Many spendings in a month**: all of the month's spendings are listed in the one table (no
  paging within a month), and totals always cover all of them.
- **Data changed elsewhere**: if a spending, category or account was changed or removed in
  another tab, the action reports a short "this item no longer exists" style message and the
  screen refreshes to the current data.
- **Session expiry**: if the session ends while the user is working, the next action shows the
  signed-out screen rather than an error; an unsaved row is lost.
- **Loading and errors**: while data loads the screen shows a loading state; if data cannot be
  loaded, a short message with a retry option is shown instead of empty-looking data.

## Requirements *(mandatory)*

### Functional Requirements

**Layout and sign-in**

- **FR-001**: The app MUST show a header with the app name, Sign in when signed out, and Sign
  out plus a user icon when signed in.
- **FR-002**: Signed-out visitors MUST see a "You're signed out" message with a Sign in button
  and MUST NOT see any spending, category or account data.
- **FR-003**: The screens MUST match the layout, wording and visual style of the design canvas
  artboards "Spendings", "Add in the table" and "Settings window", except where this spec adds
  or changes behaviour.

**Month navigation**

- **FR-010**: The Spendings screen MUST show a year with previous and next arrows and the twelve
  months of that year, each with the user's total for that month or a dash when it has none.
- **FR-011**: Months after the current month MUST be shown greyed out and MUST NOT be
  selectable; the next-year arrow MUST be unavailable while the current year is shown.
- **FR-012**: On opening, the current month of the current year MUST be selected. "Current" is
  determined in the app's configured time zone.

**Spendings table**

- **FR-020**: The table MUST list every spending of the selected month with its date, merchant,
  category, account and amount, newest date first, with the most recently recorded first among
  spendings on the same date.
- **FR-021**: The table footer MUST show the number of spendings and their total for the month.
- **FR-022**: A month with no spendings MUST show "No spendings recorded in [Month Year]."
  instead of rows.
- **FR-023**: A card MUST show the selected month's total and spending count.
- **FR-024**: Every column of the table, including the merchant column, MUST be wide enough to
  read its placeholder and typical values (at least about 160 px for the merchant). When the
  window is too narrow for the table and the summary cards side by side, the cards MUST move
  below the table; when it is too narrow for the table alone, the table MUST scroll sideways
  rather than squeeze its columns.

**Adding and editing spendings**

- **FR-030**: A + button in the table header MUST open a new editable row at the top of the
  table, with the date set to today when the current month is shown and to the 1st of the shown
  month otherwise, and its calendar open; picking a day moves focus to the merchant field.
  Pressing Enter on the Spendings screen while nothing has focus and no dialog is open MUST do
  the same as +.
- **FR-031**: Hovering a spending row, or moving keyboard focus into it, MUST reveal edit and
  remove buttons for that row.
- **FR-032**: Edit MUST turn the row into an editable row filled with the spending's current
  values; it follows the same rules as adding.
- **FR-033**: Only one row MAY be editable at a time.
- **FR-034**: The date field MUST open a calendar with previous and next month arrows, a "Today"
  shortcut, and days after today unavailable. The calendar MUST always show in full: it opens
  below the field, or above it when the window has no room below, stays inside the window, and is
  never cut off by the table.
- **FR-035**: Category and account fields MUST let the user pick from a dropdown of their
  existing items or type a name. A typed name that does not match an existing item (ignoring
  letter case) MUST be created as a new category or account when the spending is saved.
- **FR-036**: The check mark MUST save the row and X (or Escape) MUST cancel it without changes.
- **FR-037**: Before saving, every field MUST be checked; if any is missing or invalid nothing is
  saved, the row stays open with what was typed, and a message names the problem using the
  wording in User Story 2.
- **FR-038**: After a successful save, the view MUST switch to the year and month of the saved
  spending's date, and all totals, the table and the By category card MUST reflect the change
  without a full page reload.
- **FR-039**: A save MUST happen at most once per user action, even if the user clicks or presses
  Enter repeatedly; a retried save MUST NOT create a duplicate spending.
- **FR-040**: If the server refuses or cannot be reached, the user MUST see a short message
  explaining the problem, and what they typed MUST be kept.

**Autofill and keyboard entry**

- **FR-050**: As the user types in the merchant, category or account field, the field MUST
  autofill with the user's most frequently used value whose name starts with the typed text
  (ignoring letter case), showing the untyped remainder as a selected suggestion.
- **FR-051**: The dropdown MUST list all of the user's values that match the typed text, ordered
  by how many of the user's spendings use them (highest first, ties alphabetical). Values whose
  name contains the typed text but does not start with it MAY be listed after prefix matches.
  Like the calendar, the dropdown MUST always show in full, at least as wide as its field.
- **FR-057**: Past merchants are the distinct merchant names on the user's spendings, compared
  ignoring letter case and surrounding spaces; the suggested spelling is the one used on the
  most recent spending.
- **FR-058**: When the user accepts a past merchant, the category and account fields MUST be
  filled with the category and account of that merchant's most recent spending, but only where
  those fields are empty. Filled-in values MUST stay editable and are not saved until the user
  saves the row.
- **FR-052**: Tab and Enter MUST accept the current suggestion; Backspace MUST remove an
  unaccepted suggestion; arrow keys MUST move through the dropdown options.
- **FR-053**: Tab MUST move focus to the next field in the order date, merchant, category,
  account, amount, and Shift+Tab to the previous one. Right and Left arrows MUST do the same once
  the cursor is at that end of the field's text, or the whole text is selected; inside the text
  they move the cursor as usual. A field reached by an arrow has its whole text selected.
- **FR-054**: Enter in any field MUST save the spending when every field is complete, and
  otherwise move focus to the next field.
- **FR-055**: Tab or Enter in the last field (amount) MUST save the spending when every field is
  complete; otherwise it MUST show the validation message and move focus to the first
  incomplete field.
- **FR-056**: Escape MUST close an open dropdown or calendar first, and cancel the row only when
  nothing is open, from any field including the date field.

**Deleting spendings**

- **FR-060**: The remove button MUST delete the spending immediately, with no confirmation step
  and no undo.
- **FR-061**: After a delete, the table, footer, month line and cards MUST update; if the delete
  fails, the row MUST stay and the user MUST see a short message saying it was not deleted.
- **FR-062**: The remove button MUST be unavailable while the row is in edit mode, and one
  click MUST send at most one delete.

**By category**

- **FR-070**: The By category card MUST offer a Table / Pie toggle; the chosen view MUST be kept
  when the month changes.
- **FR-071**: Table view MUST list every category with spending in the selected month, and only
  those, with its amount and share of the month's total (as a whole-number percentage), largest
  first, plus a total row.
- **FR-072**: Pie view MUST show a donut chart with one slice for every category with spending
  in the selected month, and only those, with no cap on the number of slices and no "Everything
  else" grouping. A legend MUST give each slice's name, amount and share, largest first.
  Neighbouring slices MUST have distinguishable colours, and slices MUST be identifiable by more
  than colour alone (legend order and labels, and the slice's name and amount on hover or
  focus).
- **FR-073**: A month with no spendings MUST show a short empty message in the card.

**Settings**

- **FR-080**: The user icon MUST open a Settings window with sections listed on the left:
  Categories and Accounts. The window MUST close with its close button, Escape, or a click
  outside it.
- **FR-081**: Categories MUST list the 17 defaults (marked as defaults, not removable) and the
  user's own categories, each with how many spendings use it, and MUST let the user add and
  remove their own categories.
- **FR-082**: Accounts MUST list the user's accounts with how many spendings use each, and MUST
  let the user add and remove accounts.
- **FR-083**: Adding a category or account MUST be refused with a message when the name is empty
  ("Type a name for the category." / "Type a name for the account.") or matches an existing item
  ignoring letter case ("“[Name]” already exists.").
- **FR-084**: Removing a category or account that spendings use MUST be refused with a message
  giving how many spendings use it, and removing the user's last account MUST be refused with
  "You need at least one account."
- **FR-086**: The server MUST refuse to delete a user's only account as a business-rule failure
  with a stable, documented error, changing nothing. This is a backward-compatible addition to
  feature 002's interface (a new refusal reason on an existing operation).
- **FR-085**: Changes in Settings MUST be reflected in the entry dropdowns and Spendings screen
  without a full page reload.

**General**

- **FR-090**: All amounts MUST be shown in the app's single currency with two decimals (month
  line totals MAY be rounded to whole units, as in the design).
- **FR-091**: Every control MUST be usable with the keyboard alone and labelled for screen
  readers; hover-only controls MUST also appear on keyboard focus.
- **FR-092**: While data loads, the screen MUST show a loading state; when data cannot be
  loaded, it MUST show a message with a way to retry.
- **FR-093**: Screens showing personal data MUST NOT be indexed by search engines.

### Key Entities *(include if feature involves data)*

All entities come from feature 002; this feature only shows and edits them.

- **Spending**: date, merchant, category, account, amount. Shown in the table, edited inline.
- **Category**: name, whether it is a default, how many spendings use it (used for autofill
  ranking and the removal rule).
- **Account**: name, how many spendings use it (used for autofill ranking and the removal rule).
- **Month summary** (derived): for a year and month, the total and number of spendings; and for
  a month, the total per category. Shown in the month line, total card, footer and By category
  card.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: A user can record a spending whose category and account they have used before
  using only the keyboard (no mouse) in under 15 seconds.
- **SC-002**: When typing a merchant, category or account the user has used before, the intended item is
  autofilled after typing at most 3 characters in at least 90% of entries.
- **SC-003**: After any add, edit or delete, the table, footer, month line, total card and By
  category card all show the updated figures within 1 second, with no full page reload.
- **SC-004**: Totals shown on screen match the sum of the stored spending amounts to the cent in
  100% of checked months.
- **SC-005**: Switching to another month shows its spendings and breakdown within 1 second for a
  user with 10,000 spendings in total.
- **SC-006**: Every user-visible flow in this spec (view month, add, edit, delete, keyboard entry
  with autofill, by-category views, Settings add and remove, sign in and out) is covered by an
  automated end-to-end test that passes.
- **SC-007**: No spending, category or account data is visible to a signed-out visitor in any
  tested flow.
- **SC-008**: The screens pass an automated accessibility check with no serious or critical
  issues.

## Assumptions

- **Design is the source for look and wording**: colours, type, spacing and messages follow the
  design canvas; this spec adds edit, delete, autofill and keyboard behaviour not shown there.
- **Inline editing**: editing reuses the inline row used for adding; there is no separate edit
  page or dialog.
- **One-click delete**: removing a spending deletes it at once, with no confirmation and no
  undo. An accidental delete is fixed by entering the spending again.
- **Frequency means spending count**: "frequently used" ranks merchants, categories and
  accounts by how many of the user's spendings use them in total, not by recency.
- **Autofill matches the start of names**: autofill fills in only values whose name starts with
  the typed text; the dropdown may also show values that contain it elsewhere.
- **Merchant fill-in uses the latest spending**: the category and account filled in for a past
  merchant come from that merchant's most recent spending, not its most common pair.
- **No renaming in Settings**: the design offers add and remove only; renaming categories and
  accounts is out of scope here even though the server supports it.
- **No search or filters**: the Spendings screen is organised by month; searching by merchant,
  filtering by category or account, and sorting by other columns are out of scope.
- **Desktop first**: the layout is designed for desktop and laptop browsers; it stays usable at
  tablet and phone widths (content stacks, the table scrolls sideways) but phone-specific design
  is out of scope.
- **Single currency and time zone**: amounts use the one currency and "today" uses the one time
  zone set on the server (feature 002).
- **Out of scope**: income, budgets, imports, exporting, multiple years at once, and reports
  beyond the month line and By category card.

## Dependencies

- **Feature 001**: sign-in, sign-out and session handling are reused unchanged.
- **Feature 002**: spendings, categories and accounts on the server, including spending counts
  per category and account, the 17 defaults, duplicate and removal rules, and idempotent create.
- **Possible server additions**: the month line needs totals for twelve months at once and the
  By category card needs per-category totals for a month. Feature 002 deliberately left out
  totals. If showing these from the existing interface cannot meet SC-005, the plan may add
  read-only summary operations to the server; any such addition must be backward compatible.
- **Merchant suggestions**: feature 002 has no list of past merchants with their use counts and
  latest category and account. The plan must decide how the screens get this (most likely a
  read-only, backward-compatible server addition, since a user may have thousands of spendings).
- **Last-account rule**: "You can't remove your last account" is not a server rule in feature
  002; this feature adds it to the server (FR-086), so the backend changes here as well as the
  screens.
