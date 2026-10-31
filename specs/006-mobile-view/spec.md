# Feature Specification: Mobile View

**Feature Branch**: `006-mobile-view`

**Created**: 2026-10-31

**Status**: Implemented and deployed 2026-10-31: hl-budget-frontend#7 (`17-ac055d4`, via hl-helm#26) is running on the homelab (Argo CD Synced/Healthy). No backend change. Gates green; 35 desktop and 27 phone e2e journeys pass locally, and the 27 phone journeys pass against https://budget.valyalkin.com; desktop screenshots at 1280px and 1440px are identical to the previous `main`. Pending: a hands-on check on a real iPhone (quickstart §3 steps 8–9)

**Input**: User description: "I would like to work on the mobile view. When the user accesses the app from mobile, the layout and behaviour should be slightly different. IMPORTANT! Current desktop viewport behaviour MUST NOT change. Design of mobile view is available here: https://claude.ai/artifact/QTxGKQCY8eE5SUq7387QCZ#page-dcb81aee8d83 Build the mobile (phone-width, ~390px) version of HL-Budget [...] Design for touch: every tappable target at least 44px, 16px input text so iOS doesn't zoom, no hover-only interactions. [Full description: Spendings screen with Month and Year dropdowns, total, Spendings / By category switch, compact list and floating Add button; one full-screen Edit / Add screen with an inline calendar, type-or-pick category and account, validation and a bottom-sheet delete confirmation; an account menu bottom sheet (Settings, Import CSV, Sign out); a full-screen Settings with Categories | Accounts tabs; a two-step Import CSV (choose file, review list with a Fix row screen).]"

## Scope

This feature gives HL Budget a phone layout. When the app is opened on a phone-width screen,
the user gets screens designed for touch: a compact list instead of a table, full-screen forms
instead of inline rows and windows, and bottom sheets instead of hover buttons and popovers.
Everything the desktop screens can do (view a month, add, edit and delete spendings, see the
breakdown by category, manage categories and accounts, import a CSV file, sign out) can be done
on the phone.

The look, layout and wording follow the design canvas "HL-Budget"
(https://claude.ai/artifact/QTxGKQCY8eE5SUq7387QCZ), the artboards under "HL-Budget — mobile":
"Mobile — Spendings", "By category (table)", "By category (pie)", "Account menu", "Edit
spending", "Delete spending", "Add spending", "Pick a date", "Settings: categories",
"Settings: accounts", "Import CSV", "Review import" and "Fix an import row". Where this spec and
the design differ, this spec wins.

**The desktop layout does not change.** Every screen, behaviour, wording and keyboard rule
defined by features 003 (Spending Screens) and 004 (CSV Import) stays exactly as it is on
screens wider than the phone breakpoint. The phone layout is an alternative presentation of the
same app, same address, same account and same data.

The phone layout follows the same business rules as the desktop: what counts as a valid
spending, how new category and account names are created, which categories and accounts can be
removed, and how a CSV import is read and saved (all or nothing). It deliberately differs from
the desktop in these ways, each requested by the user:

- Deleting a spending asks for confirmation in a bottom sheet (the desktop deletes at once).
- The pie chart shows the top 6 categories plus "Everything else" (the desktop shows every
  category).
- The month is picked from Month and Year dropdowns (the desktop has a year line with arrows and
  twelve month buttons with totals).
- There is no drag-and-drop for CSV files and no keyboard-driven entry (Tab/Enter autofill).

No server changes are expected: the phone layout uses the data and operations the desktop
already uses.

## Clarifications

### Session 2026-10-31

- Q: Which devices get the phone layout? → A: Phones only: below 640px wide. Tablets and phones
  held sideways keep the desktop layout (Edge Cases, FR-001).
- Q: The description lists `10/27/2026` (month first) as a supported import date, but the
  desktop import reads year-last dates day first. Which rule applies on the phone? → A: The
  same as the desktop, day first. `10/27/2026` shows as "No date" and is fixed in the review
  (Edge Cases, FR-091).
- Q: What should the phone back gesture or browser Back do on a full-screen screen or bottom
  sheet? → A: Close the topmost screen or sheet without saving, like its X or Cancel; reloading
  the page returns to the Spendings list (Edge Cases, FR-103).
- Q: Should "Save row" on the Fix row screen accept a partly fixed row? → A: No. It works only
  when the whole row is valid; otherwise it shows the same inline messages as adding a spending
  and keeps the screen open (User Story 6, FR-094).
- Q: Should the phone's Merchant field suggest past merchants and fill in category and account,
  as on the desktop? → A: Yes. Past merchants are suggested most used first; picking one fills
  in the category and account of that merchant's most recent spending when they are still empty
  (User Story 2, FR-046).

## User Scenarios & Testing *(mandatory)*

### User Story 1 - See my spendings for a month on the phone (Priority: P1)

The user opens HL Budget on their phone and lands on the Spendings screen for the current
month. The header shows the logo and "HL-Budget" on the left and a user icon on the right.
Below are two dropdowns side by side, Month and Year, then the month's total in large figures
with the number of spendings, then a switch "Spendings | By category". The list shows one row
per spending with only its date (for example "Oct 27"), category and amount, newest first.

**Why this priority**: Seeing where the money went this month is the first thing the user
does on the phone, and every other story starts from this screen.

**Independent Test**: With spendings recorded in September and October of the current year,
open the app at 390px width, check October is selected and its list, total and count match the
recorded data, then pick September in the Month dropdown and check the same for September.

**Acceptance Scenarios**:

1. **Given** a signed-in user on a phone-width screen with spendings this month, **When** they
   open the app, **Then** the Month dropdown shows the current month, the Year dropdown the
   current year, the total and count ("14 spendings") match the month, and the list shows each
   spending's date, category and amount, newest date first.
2. **Given** the Month dropdown while the current year is selected, **When** the user opens it,
   **Then** months after the current month are shown but cannot be chosen.
3. **Given** the Month or Year dropdown, **When** the user chooses a different value, **Then**
   the list, total and count reload for the chosen month and year.
4. **Given** a past year with a later month selected (for example December 2025), **When** the
   user switches the year to the current year, **Then** the month changes to the current month
   if the chosen month would be in the future.
5. **Given** a month with no spendings, **When** it is selected, **Then** the list shows "No
   spendings in [Month Year]." and the total shows $0.00 with "0 spendings".
6. **Given** a long list, **When** the user scrolls, **Then** the header stays reachable at the
   top and the floating "Add spending" button stays visible at the bottom right.
7. **Given** a signed-out visitor on a phone, **When** they open the app, **Then** they see
   "You're signed out" with "Sign in to see and record your spendings." and a Sign in button.

---

### User Story 2 - Add a spending on the phone (Priority: P1)

The user taps the floating "Add spending" button. A full-screen "New spending" screen opens
with the amount field first, in large figures, so the number keypad comes up. Below are Date,
Merchant, Category and Account. Date opens a calendar inside the screen. Category and Account
can be typed or picked from a list. The user taps "Add spending" at the bottom, and is back on
the list with the new spending shown in its month.

**Why this priority**: Recording a spending on the go, right after paying, is the main reason
to use the phone at all.

**Independent Test**: At 390px width, tap "Add spending", enter 12.40, keep today's date, type a
merchant, pick an existing category and account, tap "Add spending", and check the spending
appears in the list and on the server with the exact values entered.

**Acceptance Scenarios**:

1. **Given** the Spendings list, **When** the user taps "Add spending", **Then** a full-screen
   "New spending" screen opens with an X at the top left, fields in the order Amount (CAD),
   Date, Merchant, Category, Account, and an "Add spending" button fixed at the bottom.
2. **Given** the New spending screen, **When** it opens, **Then** the date is today when the
   current month is shown, otherwise the 1st of the shown month, and the other fields are empty.
3. **Given** the amount field, **When** the user taps it, **Then** the phone shows a numeric
   keypad with a decimal point.
4. **Given** the Date field, **When** the user taps it, **Then** a calendar opens inside the
   screen showing the month of the current date value, with previous and next month arrows, a
   "Today" shortcut and a Close button; days after today cannot be picked; picking a day sets
   the date and closes the calendar.
5. **Given** the Category or Account field, **When** the user taps it, **Then** they can type a
   name or pick one of their existing names from a list that narrows as they type.
6. **Given** the user has recorded "Neighbourhood Market" most recently in "Groceries" on
   "Visa", **When** they type "nei" in Merchant and pick "Neighbourhood Market" from the
   suggestions, **Then** Category becomes "Groceries" and Account "Visa", unless those fields
   already hold a value, which is kept.
7. **Given** a category or account name that does not exist yet (ignoring letter case),
   **When** the user saves, **Then** it is created and the spending is saved with it; a name
   that matches an existing one in any letter case uses the existing one.
8. **Given** a missing or invalid field, **When** the user taps "Add spending", **Then** nothing
   is saved, everything typed is kept, and an inline message names the first problem in field
   order: "Enter an amount greater than zero.", "Pick a date.", "Add a merchant name.",
   "Choose or type a category." or "Choose or type an account."
9. **Given** a complete form, **When** the user taps "Add spending", **Then** the spending is
   saved once, the screen closes, and the list shows the month and year of the spending's date
   with the new spending in it.
10. **Given** an open New spending screen, **When** the user taps X, **Then** the screen closes
   and nothing is saved.
11. **Given** a save that the server refuses or that fails to reach the server, **When** it
    happens, **Then** the screen stays open with everything typed and shows a short message
    explaining what went wrong.

---

### User Story 3 - Edit or delete a spending on the phone (Priority: P1)

The user taps a row in the list. The same full-screen screen opens, titled "Edit spending",
with the spending's values filled in, a "Save changes" button at the bottom, and a red-outlined
"Delete spending" button. Deleting asks for confirmation in a bottom sheet first.

**Why this priority**: Fixing a typo or removing a wrong entry is part of keeping the record
correct; without it the phone is add-only.

**Independent Test**: At 390px width, tap a spending, change its amount, tap "Save changes" and
check the list and server show the new amount; tap it again, tap "Delete spending", confirm,
and check it is gone and "Deleted …" is shown.

**Acceptance Scenarios**:

1. **Given** the Spendings list, **When** the user taps a row, **Then** the "Edit spending"
   screen opens with that spending's amount, date, merchant, category and account filled in, a
   "Save changes" button at the bottom and a red-outlined "Delete spending" button below the
   fields.
2. **Given** an edited, valid form, **When** the user taps "Save changes", **Then** the change
   is saved, the screen closes, and the list shows the month and year of the spending's date.
3. **Given** the Edit spending screen, **When** the user taps X, **Then** the screen closes and
   no change is saved.
4. **Given** the Edit spending screen, **When** the user taps "Delete spending", **Then** a
   bottom sheet slides up titled "Delete this spending?" showing the merchant, amount and date
   (for example "Neighbourhood Market · $64.38 · Oct 27, 2026"), a red "Delete spending" button
   and a Cancel button.
5. **Given** the delete sheet, **When** the user taps Cancel or the dimmed area behind it,
   **Then** the sheet closes and the spending is kept, with the Edit screen still open.
6. **Given** the delete sheet, **When** the user taps "Delete spending", **Then** the spending
   is deleted, the user returns to the list, and a note shows "Deleted [merchant] · [amount]".
   The note disappears on the next action.
7. **Given** a delete that fails, **When** it happens, **Then** the spending is kept and the
   user sees a short message explaining what went wrong.

---

### User Story 4 - See the month by category on the phone (Priority: P2)

On the Spendings screen the user switches to "By category". A "Table | Pie" toggle appears.
Table lists each category with its amount and share, and a total row. Pie shows a donut chart
of the top 6 categories plus "Everything else", with a legend of amounts and shares.

**Why this priority**: The breakdown answers "where did the money go?", but recording and
fixing spendings matter more on the phone.

**Independent Test**: With October spendings in 8 categories, switch to By category, check the
table lists all 8 with amounts, shares and a total equal to the month total; switch to Pie and
check it shows 6 named slices plus "Everything else" whose amount equals the other 2 categories.

**Acceptance Scenarios**:

1. **Given** the Spendings screen, **When** the user taps "By category", **Then** the list is
   replaced by the breakdown with a "Table | Pie" toggle; the month dropdowns and total stay.
2. **Given** Table view, **When** it shows, **Then** it lists every category with spending in
   the month, highest amount first, each with its amount and share of the month (in %), and a
   final "Total" row with the month total and 100%.
3. **Given** Pie view with more than 6 categories, **When** it shows, **Then** the donut has one
   slice for each of the 6 highest categories and one "Everything else" slice for the rest, the
   month total in the middle, and a legend listing each slice's name, amount and share.
4. **Given** Pie view with 6 or fewer categories, **When** it shows, **Then** every category has
   its own slice and there is no "Everything else".
5. **Given** a month with no spendings, **When** By category is shown, **Then** it shows
   "Nothing to break down in [Month Year]." instead of the table or chart.
6. **Given** the user chose Pie (or Table) and By category, **When** they change the month,
   **Then** the same view stays selected.

---

### User Story 5 - Account menu, Settings and signing out on the phone (Priority: P2)

The user taps the user icon. A bottom sheet offers Settings, Import CSV and Sign out. Settings
opens a full-screen screen with Categories and Accounts tabs where the user adds and removes
their own categories and accounts. Sign out signs them out and shows the signed-out screen.

**Why this priority**: Managing categories and accounts is occasional; most of it also happens
implicitly by typing new names when adding a spending.

**Independent Test**: At 390px width, open the menu, open Settings, add the category "Coffee",
try adding "coffee" (refused), remove "Coffee"; on Accounts, add "Savings", check it shows "No
spendings yet", remove it; try removing an account in use (refused); sign out and check the
signed-out screen appears.

**Acceptance Scenarios**:

1. **Given** the Spendings screen, **When** the user taps the user icon, **Then** a bottom sheet
   opens with Settings, Import CSV and Sign out; tapping the dimmed area behind it closes it.
2. **Given** the menu, **When** the user taps Settings, **Then** a full-screen "Settings" screen
   opens with an X to close and a "Categories | Accounts" tab switch, Categories selected.
3. **Given** the Categories tab, **When** it shows, **Then** it has a short explanation, a
   "New category" input with an "Add category" button below it, a "Your categories · N" list
   with a remove button for each (or "No custom categories yet."), and "Default categories · 17"
   shown as tags with no remove button.
4. **Given** a new category name, **When** the user taps "Add category", **Then** it is added to
   "Your categories" and the input clears; an empty name is refused with "Type a name for the
   category." and a name matching any existing category ignoring letter case with "“[Name]”
   already exists."
5. **Given** a custom category used by spendings, **When** the user taps its remove button,
   **Then** it is kept and the message says "[Name] is used by N spendings, so it can't be
   removed."
6. **Given** the Accounts tab, **When** it shows, **Then** it has a short explanation, a "New
   account" input with an "Add account" button, and every account with how many spendings it
   has ("12 spendings recorded" or "No spendings yet") and a remove button.
7. **Given** an account, **When** the user tries to remove it while it has spendings, **Then**
   it is kept with "[Name] has N spendings, so it can't be removed."; **when** it is the last
   account, **Then** it is kept with "You need at least one account."
8. **Given** the menu, **When** the user taps Sign out, **Then** they are signed out and see the
   "You're signed out" screen with a Sign in button.

---

### User Story 6 - Import a CSV file on the phone (Priority: P3)

From the account menu the user opens Import CSV, taps a large "Choose file" button and picks a
bank export from the phone. A review list shows every row; rows with missing or unreadable
values are marked. The user taps each marked row, fixes it on a "Fix row" screen (or removes
it), then taps "Add N spendings".

**Why this priority**: Importing statements is a batch job that is easier at a desk; on the
phone it is a convenience, so it comes last.

**Independent Test**: At 390px width, import a CSV with 10 rows of which 4 have a missing or
unreadable value; check the banner says "4 rows need attention" and the add button is disabled;
fix 3 rows and remove 1; check the banner says "All rows look good"; tap "Add 9 spendings" and
check all 9 are saved, the list shows the latest imported month, and "Added 9 spendings from
[file]" is shown.

**Acceptance Scenarios**:

1. **Given** the account menu, **When** the user taps Import CSV, **Then** a full-screen
   "Import CSV" screen opens with an X to close, a short explanation, a large "Choose file"
   button that opens the phone's file picker limited to CSV files, and a note: columns for
   date, merchant, category, account and amount, with a header row, in any order, and anything
   missing can be filled in on the next screen.
2. **Given** the file picker, **When** the user picks a file that is not a CSV, is empty, has no
   data rows, cannot be read or is over the size limit, **Then** the screen stays open and shows
   the matching error message, and nothing is imported.
3. **Given** a readable CSV, **When** it is picked, **Then** a "Review import" screen shows the
   file name and row count, a banner ("N rows need attention" or "All rows look good"), the hint
   "Tap a row to check or fix it.", and one row per data row in file order with the merchant on
   the first line, "date · category · account" on the second, and the amount on the right.
4. **Given** a row with a missing or invalid value, **When** it is listed, **Then** that part is
   shown in amber as "No date", "Merchant missing", "No category", "No account" or "No amount",
   and the row shows a warning icon; a valid row shows a check icon.
5. **Given** the review list, **When** the user taps a row, **Then** the edit screen opens
   titled "Fix row" with the row's values, a "Save row" button at the bottom, and a "Remove row"
   button; X returns to the review without changes.
6. **Given** the Fix row screen, **When** the user completes the row and taps "Save row",
   **Then** the row is updated in the review (nothing is saved to the server yet) and the
   review list shows again with the banner and counts updated. **When** any field is still
   missing or invalid, **Then** the row is not updated, the screen stays open with everything
   typed, and it shows the same inline message as adding a spending.
7. **Given** the Fix row screen, **When** the user taps "Remove row", **Then** the row is taken
   out of the review at once, with no confirmation, and the review list shows again; when no
   rows remain it shows "No rows left to import."
8. **Given** any row still needs attention or no rows remain, **When** the user looks at the
   bottom button, **Then** it is disabled and reads "Add spendings"; when every row is valid it
   is enabled and reads "Add N spendings" (or "Add 1 spending").
9. **Given** all rows valid, **When** the user taps "Add N spendings", **Then** every row is
   saved once, all or nothing, with any new category or account names created; the user lands
   on the Spendings list showing the year and month of the latest imported date, with the note
   "Added N spendings from [file name]".
10. **Given** an add that fails or is refused, **When** it happens, **Then** nothing is saved,
    the review stays with every row and fix, and a short message explains what went wrong.
11. **Given** the Review import screen, **When** the user taps its back button, **Then** the
    rows and fixes are discarded, nothing is saved, and the Import CSV screen shows again.

---

### Edge Cases

- **Which layout**: the phone layout is used when the visible width of the browser is below the
  phone breakpoint of 640 CSS pixels, and the desktop layout at 640px and wider. Phones in
  portrait get the phone layout; tablets and phones held sideways get the desktop layout.
- **Width changes while open** (rotating the phone, resizing a desktop window across the
  breakpoint): the app switches layout and keeps the selected month and year. An unsaved New
  spending, Edit spending or Fix row screen, an open menu or sheet, or an in-progress import
  review may be closed without saving.
- **Phone back gesture or browser Back**: on a full-screen screen or a bottom sheet, it closes
  that screen or sheet without saving, the same as its X or Cancel (on Review import it returns
  to the Import CSV screen); on the Spendings screen it behaves as the browser normally does.
  Full-screen screens have no address of their own: reloading the page on any of them shows the
  Spendings list, and anything unsaved is lost.
- **Future dates**: the calendar does not allow days after today; an imported date after today
  needs attention: the row gets a warning icon and the date is shown in amber.
- **Amount input**: a negative amount typed by hand is refused ("Enter an amount greater than
  zero."); amounts with more than 2 decimal places are refused by the same rule as the desktop.
- **Imported amounts**: currency symbols, thousands separators and spaces are removed; a
  negative amount (for example "-18.90") is read as the positive amount 18.90, a normal
  spending.
- **Imported dates**: `2026-10-28` is read year-month-day. A date with the year last, such as
  `27/10/2026`, is always read day first (DD/MM/YYYY), exactly as the desktop import reads it,
  so one file gives the same result on both layouts. A month-first date such as `10/27/2026`
  cannot be read: the row shows "No date" and needs fixing in the review.
- **Imported CSV quoting**: quoted values with commas, double quotes or line breaks inside are
  read as one value (for example `"Lakeshore Fuel, Hwy 7"`); blank lines and a leading
  byte-order mark are ignored.
- **Header names**: columns are matched by header name in any order and letter case, with the
  same alternative names the desktop import accepts (for example "Description" for merchant).
- **Import file limits**: the same limits as the desktop import (1 MB, 1,000 data rows), with
  the same messages.
- **Duplicates**: imported rows are not checked against existing spendings or each other, as on
  the desktop.
- **Category and account matching on import**: a value matching an existing name in any letter
  case is shown with the existing name; new names are created once each when the rows are added.
- **Tapping twice**: tapping a save, delete or add button twice quickly saves or deletes only
  once.
- **Slow or failed loading**: while the month loads the list shows a loading state; when it
  cannot be loaded, a message with a way to retry is shown.
- **Long names**: long merchant, category and account names are shortened with an ellipsis in
  lists and never push the amount off the screen.
- **Large totals**: a month total of $100,000.00 or more still fits on one line at 390px.
- **Empty account list in Settings**: cannot happen (the last account cannot be removed).
- **Years in the Year dropdown**: from the current year back to the earliest year with a
  spending, and at least the current year.

## Requirements *(mandatory)*

### Functional Requirements

**Layout selection and desktop protection**

- **FR-001**: The app MUST show the phone layout when the browser's visible width is below the
  phone breakpoint (see Edge Cases) and the desktop layout otherwise, at the same address, with
  no separate sign-in or setting.
- **FR-002**: The desktop layout MUST remain exactly as specified by features 003 and 004: same
  screens, wording, layout, keyboard behaviour and rules. No desktop behaviour may be added,
  removed or changed by this feature.
- **FR-003**: The phone screens MUST match the layout, wording and visual style of the mobile
  artboards of the design canvas, except where this spec adds or changes behaviour.
- **FR-004**: Every tappable control in the phone layout MUST have a touch target of at least
  44 by 44 CSS pixels.
- **FR-005**: Every text input and dropdown in the phone layout MUST use text of at least 16 CSS
  pixels, so phones do not zoom in when it is tapped.
- **FR-006**: No action in the phone layout MAY depend on hovering; every action MUST be
  reachable by tapping.
- **FR-007**: The phone layout MUST fit 320 to 639 CSS pixels wide without sideways scrolling.

**Spendings screen**

- **FR-010**: The header MUST show the logo and "HL-Budget" on the left and a user icon button
  on the right that opens the account menu.
- **FR-011**: The screen MUST show a Month dropdown and a Year dropdown side by side; both MUST
  use the phone's own dropdown picker.
- **FR-012**: In the current year, months after the current month MUST be shown disabled.
- **FR-013**: The Year dropdown MUST list years from the current year back to the earliest year
  that has a spending, and at least the current year.
- **FR-014**: Changing either dropdown MUST reload the list, total, count and breakdown for the
  chosen month; choosing the current year with a later month selected MUST change the month to
  the current month.
- **FR-015**: On opening, the current month of the current year MUST be selected.
- **FR-016**: Below the dropdowns the screen MUST show the month's total in large monospaced
  figures and the number of spendings ("1 spending", "14 spendings").
- **FR-017**: The screen MUST show a two-option switch "Spendings | By category"; Spendings is
  selected on opening.
- **FR-018**: The Spendings view MUST list every spending of the month, newest date first, each
  row showing only the date as "Mon D" (for example "Oct 27"), the category and the amount;
  tapping a row MUST open the Edit spending screen for it.
- **FR-019**: A month with no spendings MUST show "No spendings in [Month Year]."
- **FR-020**: A floating "Add spending" button MUST stay visible at the bottom right while the
  list scrolls, MUST NOT cover the last row when scrolled to the end, and MUST open the New
  spending screen.
- **FR-021**: Each row MUST be announced to screen readers with its date, merchant, category and
  amount, even though the merchant is not shown.

**By category view**

- **FR-030**: The By category view MUST offer a "Table | Pie" toggle; the chosen view MUST be
  kept when the month changes.
- **FR-031**: Table MUST list every category with spending in the month, highest amount first,
  with its amount and share in %, and a final "Total" row with the month total and 100%.
- **FR-032**: Pie MUST show a donut chart with one slice for each of the 6 highest categories
  and, when there are more, one "Everything else" slice for the rest; the month total MUST be
  shown in the middle; a legend MUST list each slice's name, colour, amount and share.
- **FR-033**: A month with no spendings MUST show "Nothing to break down in [Month Year]."
- **FR-034**: The chart's values MUST also be available as text (the legend) for screen readers.

**Edit / Add screen**

- **FR-040**: Adding and editing MUST use one full-screen screen with an X (close without
  saving) at the top left, a title, the fields, and an action button fixed at the bottom that
  stays visible above the phone keyboard.
- **FR-041**: The title and button MUST be "New spending" / "Add spending" when adding, "Edit
  spending" / "Save changes" when editing, and "Fix row" / "Save row" when fixing an import row.
- **FR-042**: The fields MUST be in the order Amount (CAD), Date, Merchant, Category, Account.
  Amount MUST be shown large and MUST bring up a numeric keypad with a decimal point.
- **FR-043**: A new spending's date MUST default to today when the current month is shown,
  otherwise to the 1st of the shown month.
- **FR-044**: The Date field MUST show the date as "Mon D, YYYY" and open a calendar inside the
  screen with previous and next month arrows, the month and year, the days of the week, a
  "Today" shortcut and a Close button; days after today MUST NOT be pickable; picking a day
  MUST set the date and close the calendar.
- **FR-045**: Category and Account MUST accept typed text and offer the user's existing names
  (for category: the 17 defaults and their own), narrowing as they type, most used first.
- **FR-046**: The Merchant field MUST offer the user's past merchants as they type, most used
  first; picking a past merchant MUST fill in the category and account of that merchant's most
  recent spending when those fields are still empty, as on the desktop.
- **FR-047**: Saving MUST apply the same rules as the desktop: amount greater than zero with at
  most 2 decimals, a date today or earlier, and merchant, category and account not empty and
  within their length limits. If anything is missing or invalid, nothing is saved, everything
  typed is kept, and an inline message names the first problem in field order (messages in
  User Story 2).
- **FR-048**: Saving a spending MUST create a typed category or account name that does not exist
  yet (ignoring letter case) in the same step, so nothing is created if the spending is refused.
- **FR-049**: After a successful add or save, the screen MUST close and the Spendings list MUST
  show the year and month of the spending's date, with the list, total, count and breakdown
  updated.
- **FR-050**: A save MUST happen at most once per tap, even if the button is tapped repeatedly.
- **FR-051**: If the server refuses or cannot be reached, the screen MUST stay open with
  everything typed and show a short message explaining what went wrong.

**Deleting**

- **FR-060**: The Edit spending screen MUST show a red-outlined "Delete spending" button below
  the fields; the New spending screen MUST NOT.
- **FR-061**: "Delete spending" MUST open a bottom-sheet confirmation titled "Delete this
  spending?" with the merchant, amount and date, a red "Delete spending" button and a Cancel
  button. Cancel or tapping outside the sheet MUST close it and keep the spending.
- **FR-062**: Confirming MUST delete the spending once, return to the list, and show the note
  "Deleted [merchant] · [amount]" until the next action.
- **FR-063**: If the delete fails, the spending MUST be kept and a short message shown.

**Account menu and signing out**

- **FR-070**: The user icon MUST open a bottom sheet with Settings, Import CSV and Sign out;
  tapping outside the sheet MUST close it.
- **FR-071**: Sign out MUST sign the user out and show the "You're signed out" screen with a
  Sign in button; Sign in MUST start the normal sign-in.

**Settings**

- **FR-080**: Settings MUST be a full-screen screen with an X to close and a "Categories |
  Accounts" tab switch; Categories is selected on opening.
- **FR-081**: Categories MUST show a short explanation, a "New category" input stacked above an
  "Add category" button, a "Your categories · N" list with a remove button per category (or "No
  custom categories yet."), and the 17 default categories as tags that cannot be removed.
- **FR-082**: Accounts MUST show a short explanation, a "New account" input stacked above an
  "Add account" button, and each account with how many spendings it has ("N spendings recorded"
  or "No spendings yet") and a remove button.
- **FR-083**: Adding MUST be refused with the desktop's messages when the name is empty or
  matches an existing item ignoring letter case.
- **FR-084**: Removing a category or account that spendings use MUST be refused with a message
  giving how many spendings use it; removing the last account MUST be refused with "You need
  at least one account."
- **FR-085**: Changes in Settings MUST show in the Edit / Add screen's lists and the Spendings
  screen without reloading the page.

**Import CSV**

- **FR-090**: Import CSV MUST open a full-screen screen with an X to close, a short explanation,
  a large "Choose file" button that opens the phone's file picker limited to CSV files, and the
  note on expected columns. There MUST be no drag-and-drop area.
- **FR-091**: The phone import MUST read files by the same rules as the desktop import (feature
  004): columns matched by header name in any order and letter case, standard CSV quoting,
  amounts with currency symbols, negative amounts read as positive, the same size and row
  limits, and the same error messages for files that are not CSV, cannot be read, are too large
  or have no data rows. Dates follow the rule in Edge Cases.
- **FR-092**: A value that cannot be read or is absent MUST leave that part of the row empty and
  marked, not refuse the file.
- **FR-093**: The Review import screen MUST show a back button, the title, the file name and row
  count, a banner ("N rows need attention" or "All rows look good"), the hint "Tap a row to
  check or fix it.", and every row (no paging) with the merchant, "date · category · account"
  and the amount; missing or invalid parts MUST be shown in amber with the texts in User Story
  6, and rows needing attention MUST show a warning icon.
- **FR-094**: Tapping a row MUST open the Edit / Add screen titled "Fix row" with the row's
  values; "Save row" MUST apply the same checks as adding a spending and accept only a fully
  valid row, then update the row only in the review and return to the list; the delete button MUST read "Remove row" and take the row out
  at once, with no confirmation.
- **FR-095**: The bottom button MUST be disabled and read "Add spendings" while any row needs
  attention or no rows remain, and otherwise read "Add N spendings" (or "Add 1 spending").
- **FR-096**: Adding MUST save every row, whatever its month, create any new category or account
  names once each, and be all or nothing; it MUST happen at most once per tap.
- **FR-097**: After a successful add, the Spendings list MUST show the year and month of the
  latest imported date with the note "Added N spendings from [file name]".
- **FR-098**: If adding fails, nothing MUST be saved, and the review MUST stay with every row and
  fix and show a short message.
- **FR-099**: The back button on Review import MUST discard the rows and fixes and return to the
  Import CSV screen; the file's contents MUST NOT be stored anywhere except as the spendings
  the user adds.

**General**

- **FR-100**: All amounts MUST be shown in CAD with two decimals.
- **FR-101**: Bottom sheets and full-screen screens MUST keep the screen behind them from being
  used or scrolled while open, and MUST be announced as dialogs to screen readers.
- **FR-102**: Every control MUST be labelled for screen readers, including icon-only buttons
  (user icon, X, back, remove); errors MUST be announced.
- **FR-103**: The phone back gesture and browser Back MUST close the topmost full-screen screen
  or bottom sheet without saving, as described under Edge Cases; reloading the page MUST show the
  Spendings list.
- **FR-104**: Screens showing personal data MUST NOT be indexed by search engines, and the
  phone layout MUST follow the same sign-in and per-user data rules as the desktop.

### Key Entities *(include if feature involves data)*

No new data. The phone layout shows and changes the same entities as the desktop:

- **Spending**: date, merchant, category, account, amount in CAD; belongs to one user.
- **Category**: the 17 fixed defaults plus the user's own; names unique ignoring letter case.
- **Account**: the cards and bank accounts the user pays from; at least one always exists.
- **Import row**: a row read from a CSV file while it is being reviewed; held only on the phone
  until the user adds the rows, never stored on its own.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: On a 390px-wide phone, a user can record a spending with an existing category and
  account in under 20 seconds from opening the app.
- **SC-002**: Every user flow in this spec (view a month, add, edit, delete, breakdown, menu,
  settings, sign out, import) can be completed at 320px, 390px and 430px widths with no
  sideways scrolling and no text cut off without an ellipsis.
- **SC-003**: 100% of tappable controls in the phone layout measure at least 44 by 44 CSS
  pixels, and 100% of text inputs and dropdowns use text of at least 16 CSS pixels.
- **SC-004**: Tapping any input on an iPhone does not zoom the page.
- **SC-005**: Every existing desktop end-to-end and unit check from features 003 and 004 passes
  unchanged, and desktop screens at 1280px and 1440px look the same before and after this
  feature.
- **SC-006**: Accessibility scans of every phone screen and sheet report no serious or critical
  issues.
- **SC-007**: Importing a 100-row CSV and fixing its flagged rows on the phone ends with exactly
  the reviewed rows saved, with no duplicates even when the add button is tapped repeatedly.

## Assumptions

- "Phone" means a browser whose visible width is below 640 CSS pixels (Clarifications).
- The phone layout is part of the existing web app (same address and sign-in), not a separate
  installable app; offline use is out of scope.
- Most-used ordering of category and account suggestions (FR-045) reuses the desktop's rule.
- The desktop's keyboard rules (Tab/Enter autofill, arrow keys, Escape) are not required on the
  phone layout.
- The phone import offers no sample file; the design's "Try a sample file" button is left out
  (the desktop keeps its "Download sample CSV" link).
- Year options start at the earliest year with a spending (the design hard-codes 2025).
- The "Deleted …" and "Added …" notes stay until the user's next action, as in the design.
- No server changes are needed: all data and operations already exist from features 002 to 004.
- The design canvas's sample data (merchants, amounts) is illustrative only.
