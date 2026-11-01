# Feature Specification: Income

**Feature Branch**: `009-income-tracking`

**Created**: 2026-11-01

**Status**: Draft

**Input**: User description: "Now, I would like to introduce Income tab along spendings. It will have the same mechanics as Spendings, but for income. Default income categories are Salary and Investments. The rest should be follow the same practices and UI."

## Overview

HL Budget records only money going out. This feature adds money coming in: an **Income** tab next
to **Spendings**, where the user records, views, edits, deletes and imports income, month by month,
and sees it broken down by category and by account.

The Income tab works like the Spendings tab in every respect unless this spec says otherwise. That
means the current behaviour of features 003 (desktop screens), 004 (CSV import), 006 (phone
layout), 007 (fixed colours, slice labels, categories and accounts created only in Settings) and
008 (By account), applied to income:

- A month line (desktop) or Month and Year dropdowns (phone), with the month's income total.
- A table of the month's income with an inline add row, autofill and keyboard entry, edit and
  one-click remove on the desktop; a compact list, full-screen forms and a delete confirmation on
  the phone.
- A month total card, a By category card and a By account card, each as a table or a donut.
- Import CSV, with the same review, fixing and all-or-nothing save.

What differs from Spendings:

- **Income has its own categories.** Every user starts with two fixed defaults, **Salary** and
  **Investments**, and can add their own in Settings. Income categories and spending categories are
  separate lists: a spending cannot use an income category, and an income cannot use a spending
  category.
- **Accounts are shared.** The same accounts are used for spendings and income (income is paid
  into an account, spendings are paid out of one). They are still created only in Settings.
- **"Source" instead of "Merchant".** An income entry records who paid it (for example an
  employer or a broker) in a Source field, which behaves like Merchant.

Nothing about the Spendings tab changes, apart from the tab switch that leads to Income and the
account rules that now also count income (FR-052, FR-053).

## User Scenarios & Testing *(mandatory)*

### User Story 1 - See my income for a month (Priority: P1)

The signed-in user switches from Spendings to the Income tab and sees the month's income: a month
line with each month's income total, a table of the selected month's income entries (date, source,
category, account, amount), newest first, a footer with the count and total, and a card with the
month's income total.

**Why this priority**: Seeing income is half of the feature and the screen every other story
lives on.

**Independent Test**: With income recorded through the server in September and October of the
current year, open the app, switch to Income, check October is selected and its table, footer and
card match the data, then click September and check the same.

**Acceptance Scenarios**:

1. **Given** a signed-in user on the Spendings tab, **When** they choose Income, **Then** the Income
   tab shows the same year and month that were selected on Spendings, with that month's income
   entries listed newest date first and a footer with the count (for example "3 entries") and the
   total.
2. **Given** the Income tab, **When** the user looks at the month line, **Then** each past and
   current month shows its income total (or a dash when it has none), and future months are greyed
   out, as on Spendings.
3. **Given** a month with no income, **When** the user selects it, **Then** the table shows "No
   income recorded in [Month Year]." and the card shows a total of zero and 0 entries.
4. **Given** the Income tab, **When** the user switches back to Spendings, **Then** Spendings shows
   the month that was selected on Income, and its figures are spendings only.
5. **Given** the user opens or reloads the app, **When** it loads, **Then** it opens on the
   Spendings tab for the current month, as today.
6. **Given** the Spendings tab, **When** the user looks at any total, table or breakdown, **Then**
   no income is included; and the Income tab never includes spendings.

---

### User Story 2 - Record, edit and delete income on the desktop (Priority: P1)

On the Income tab the user clicks +, gets the same editable row as on Spendings with fields date,
source, category, account and amount, and saves with the check mark, Enter or Tab on the last
field. Hovering a row shows edit and remove, which work exactly as on Spendings. Source, category
and account autofill as merchant, category and account do on Spendings.

**Why this priority**: Recording income is the core action of the feature.

**Independent Test**: With one account, open Income, click +, type "Acme Corp", pick Salary and the
account, type 5000, press Enter, and check the entry appears in the table, the totals and the
server. Edit its amount, then remove it, and check the table and totals each time.

**Acceptance Scenarios**:

1. **Given** the Income tab, **When** the user clicks + (or presses Enter with nothing focused),
   **Then** an editable row opens at the top with the date set as on Spendings (today in the
   current month, otherwise the 1st of the shown month) and its calendar open.
2. **Given** the category field, **When** the user types or opens its dropdown, **Then** only
   income categories are offered (Salary, Investments and the user's own), most used first; no
   spending category is offered.
3. **Given** the source field, **When** the user types the start of a source used before, **Then**
   it autofills the most used matching source; accepting it fills in the category and account of
   that source's most recent income entry where those fields are empty.
4. **Given** a complete row, **When** the user saves, **Then** the entry is saved once, the view
   moves to the entry's month, and the month line, table, footer, total card, By category and By
   account cards on the Income tab include it.
5. **Given** a row with a missing or invalid field, **When** the user tries to save, **Then**
   nothing is saved and a message names the first problem: "Pick a date.", "Add a source.", "Choose
   a category.", "Choose an account." or "Enter an amount greater than zero."
6. **Given** a category that is not one of the user's income categories (including a spending
   category's name, such as "Groceries"), **When** it is typed, **Then** the field is highlighted
   with "Not one of your income categories" and the entry cannot be saved.
7. **Given** an account name that does not exist, **When** it is typed, **Then** the field shows
   "Not one of your accounts" and the entry cannot be saved.
8. **Given** an income row, **When** the user clicks remove, **Then** the entry is deleted at once
   with no confirmation, as on Spendings, and every total updates.
9. **Given** a user with no accounts, **When** they start adding income, **Then** they see the same
   message as on Spendings pointing them to Settings to add an account.

---

### User Story 3 - See where the month's income came from (Priority: P2)

The Income tab shows a By category card and a By account card for the selected month, each with
its own Table / Pie toggle, fixed colours and slice labels, exactly like the cards on Spendings.

**Why this priority**: Useful insight, but income can be tracked without it, and most users have
few income categories.

**Independent Test**: With October income in Salary and Investments on two accounts, check By
category lists Salary and Investments with amount, share and a total equal to the month's income
total; switch to Pie and check two slices in the categories' colours; check the same for By
account.

**Acceptance Scenarios**:

1. **Given** By category on the Income tab, **When** a month with income is selected, **Then** each
   income category used that month is listed with its colour swatch, amount and share, largest
   first, and a total row equal to the month's income total.
2. **Given** Pie view, **When** the user hovers, focuses or taps a slice, **Then** it shows the
   category (or account) name, amount and share, as on Spendings.
3. **Given** By account on the Income tab, **When** a month with income is selected, **Then** each
   account that received income that month is listed with its colour (the same colour the account
   has on the Spendings tab), amount and share.
4. **Given** Salary and Investments, **When** the user views any month, **Then** each always has
   the same colour, for every user.
5. **Given** the Table / Pie choice for a card on the Income tab, **When** the user changes month
   or switches tabs and back, **Then** the choice is kept, separately from the Spendings cards'
   choices.
6. **Given** a month with no income, **When** the cards are shown, **Then** each shows a short
   empty message.

---

### User Story 4 - Manage income categories in Settings (Priority: P2)

Settings gains an **Income categories** section next to the existing categories (now labelled
**Spending categories**) and Accounts. It lists Salary and Investments (marked as defaults, not
removable) and the user's own income categories with how many income entries use each, and lets
the user add and remove their own.

**Why this priority**: The two defaults cover most income; custom categories (for example
"Freelance" or "Gifts") are needed but less often.

**Independent Test**: Open Settings, add an income category "Freelance", check it appears in
Settings and in the Income entry row's dropdown but not in the Spendings dropdown, record an income
in it, check removing it is refused, delete that income, then remove it.

**Acceptance Scenarios**:

1. **Given** Settings, **When** the user opens Income categories, **Then** they see Salary and
   Investments marked as defaults with no remove button, and their own income categories, each with
   a colour swatch and how many income entries use it.
2. **Given** Income categories, **When** the user adds an empty name, **Then** they see "Type a name
   for the category."; **when** the name matches an existing income category in any letter case,
   **Then** they see "“[Name]” already exists." and nothing is added.
3. **Given** Income categories, **When** the user adds a name that is also a spending category (for
   example "Other"), **Then** it is accepted: the two lists are separate.
4. **Given** a custom income category that income entries use, **When** the user tries to remove it,
   **Then** nothing is removed and they see "[Name] is used by N entries, so it can't be removed."
5. **Given** an account used only by income entries, **When** the user tries to remove it in
   Accounts, **Then** nothing is removed and they see how many spendings and income entries use it.
6. **Given** the phone layout, **When** the user opens Settings from the account menu, **Then**
   Income categories is available there with the same rules.

---

### User Story 5 - Import income from a CSV file (Priority: P3)

On the Income tab, Import CSV works as on Spendings, but the file's rows become income entries:
the review checks categories against income categories, and Add saves all rows as income at once.

**Why this priority**: Income entries are few each month (a salary, a dividend), so typing them is
quick; import helps mainly with a backlog.

**Independent Test**: On the Income tab, download the sample file, import it, check the review
shows its rows with Salary and Investments as valid categories, add them, and check they appear on
the Income tab and not on Spendings.

**Acceptance Scenarios**:

1. **Given** the Income tab, **When** the user chooses Import CSV, **Then** the import window says
   it imports income, and the downloadable sample file contains income examples (Salary and
   Investments).
2. **Given** a file with a category that is a spending category or unknown, **When** it is
   reviewed, **Then** that row is flagged "Not one of your income categories" and Add stays greyed
   out until it is fixed.
3. **Given** a valid review, **When** the user adds it, **Then** every row is saved as an income
   entry, all or nothing, retry-safe, and none of them appears on the Spendings tab.
4. **Given** the Spendings tab, **When** the user imports a file, **Then** it imports spendings, as
   today.

---

### User Story 6 - Income on the phone (Priority: P2)

On the phone layout (below 640px wide) the user switches between Spendings and Income and gets the
same phone screens for income: Month and Year dropdowns, the month's income total, a compact list
with a floating "Add income" button, full-screen Add income / Edit income forms, a delete
confirmation sheet, and the By category and By account views with the top-6 donut.

**Why this priority**: Recording a salary on the go is useful, but the desktop is where months are
reviewed.

**Independent Test**: On a phone-width screen, switch to Income, tap Add income, fill in the form,
save, and check the entry appears in the list and in By category; delete it through the
confirmation sheet.

**Acceptance Scenarios**:

1. **Given** the phone Spendings screen, **When** the user switches to Income, **Then** the screen
   shows the same month's income, with the view switch "Income | By category | By account".
2. **Given** the Income list, **When** the user taps the floating "Add income" button, **Then** a
   full-screen "Add income" form opens with Date, Source, Category, Account and Amount, and the
   category picker lists only income categories.
3. **Given** an income entry in the list, **When** the user taps it, **Then** "Edit income" opens,
   with Delete asking for confirmation in a bottom sheet, as for spendings.
4. **Given** a month with no income, **When** the list is shown, **Then** it reads "No income in
   [Month Year]."
5. **Given** the phone layout, **When** the user switches between Spendings and Income, **Then**
   the switch fits the screen width, each option is a touch target of at least 44px, and the back
   gesture does not switch tabs.

---

### Edge Cases

- **Same name in both lists**: an income category and a spending category may share a name (for
  example "Other"); they are different categories with their own colours and counts.
- **A spending category typed on Income** (or the reverse): it is not one of the field's
  categories, so it is flagged invalid, never converted or created.
- **Defaults for existing users**: users who existed before this feature get Salary and Investments
  once, on first use, never duplicated, as spending defaults were given in feature 002.
- **Account removal**: an account used by any spending or any income entry cannot be removed, and
  the message gives both counts; the "last account" rule is unchanged.
- **Renamed account**: its new name shows on both tabs, and it keeps its colour on both.
- **Account colours**: an account has one colour, used in By account on both tabs.
- **Income category colours**: Salary and Investments have fixed colours, different from each
  other; custom income categories get unused colours by the same rule as custom spending
  categories, assigned within the income categories (an income category may share a colour with a
  spending category).
- **Source spellings**: "acme corp" and "Acme Corp " count as one past source, as merchants do.
  Past sources are taken from income entries only and never mixed with merchants.
- **Switching tabs with an unsaved row**: switching tabs with an add or edit row open discards the
  row, as cancelling it would.
- **Switching tabs during an import review**: the import window is modal, so the tab cannot change
  until the review is added or cancelled.
- **Future dates, amounts, long names, double save, session expiry, data changed elsewhere,
  loading and errors**: as on Spendings.
- **Many entries in a month**: all are listed in one table, no paging, as on Spendings.

## Requirements *(mandatory)*

### Functional Requirements

**Tabs**

- **FR-001**: The signed-in screen MUST offer two tabs, Spendings and Income, on the desktop and the
  phone layouts. The app MUST open on Spendings.
- **FR-002**: Switching tabs MUST keep the selected year and month, and MUST NOT reload the page.
- **FR-003**: Each tab MUST show only its own kind of entry: no total, list, breakdown, autofill
  suggestion or count on one tab may include entries of the other kind.
- **FR-004**: The Spendings tab MUST behave exactly as before this feature, apart from FR-001 and
  the account rules in FR-052 and FR-053.

**Income entries**

- **FR-010**: An income entry MUST have a date, a source, an amount, one income category and one
  account, all required, with the same rules as a spending's date, merchant and amount (feature
  002): the date not later than today in the app's time zone, the source up to 255 characters, the
  amount positive with at most 2 decimals, in the app's single currency.
- **FR-011**: Every income entry MUST belong to one user, and a user MUST only see, change or
  delete their own; another user's entry MUST be reported as not found.
- **FR-012**: The server MUST offer create (with an optional idempotency key), list, fetch-one,
  update and delete for income entries, with the same paging, filtering and sorting as spendings,
  and the same validation and error reporting.
- **FR-013**: The category of an income entry MUST be one of the same user's income categories, and
  its account one of the same user's accounts. Saving an income entry MUST NOT create any category
  or account, even if the screen's checks are bypassed.

**Income categories**

- **FR-020**: Each user MUST have the default income categories **Salary** and **Investments** from
  their first use, without any setup step, given exactly once and never duplicated, including users
  who existed before this feature.
- **FR-021**: Salary and Investments MUST NOT be renamed or removed.
- **FR-022**: Income categories MUST be a separate list from spending categories. Names MUST be
  unique within the user's income categories (ignoring case and surrounding spaces), non-empty and
  at most 50 characters; the same name MAY exist as a spending category.
- **FR-023**: Income categories MUST be created only in Settings. Users MUST be able to add and
  remove their own; removing one that any income entry uses MUST be refused with the number of
  entries that use it.
- **FR-024**: Each income category MUST have a fixed colour: Salary and Investments each a fixed
  colour, the same for every user, different from each other; a custom income category a colour
  given on creation, unused among the user's income categories while the palette has unused
  colours, kept for its lifetime. Colours come from the existing palette and meet its 3:1 contrast
  rule.
- **FR-025**: Every income category returned MUST include how many of the user's income entries use
  it.

**Accounts**

- **FR-050**: Accounts MUST be shared by spendings and income: the same list, names and colours on
  both tabs and in Settings.
- **FR-051**: Accounts MUST still be created only in Settings.
- **FR-052**: Removing an account that any spending or any income entry uses MUST be refused,
  changing nothing, and the message MUST give how many spendings and how many income entries use
  it.
- **FR-053**: Settings MUST show for each account how many spendings and how many income entries
  use it.

**Desktop Income tab**

- **FR-060**: The Income tab MUST have the Spendings tab's layout: the month line with each month's
  income total, the table with inline add, edit and remove, the footer, the month total card, the By
  category card and the By account card.
- **FR-061**: The table's columns MUST be Date, Source, Category, Account and Amount, and its
  wording MUST say income rather than spendings (for example "No income recorded in [Month Year].",
  "3 entries").
- **FR-062**: Adding, editing, removing, autofill, keyboard entry, validation messages, one-at-a-time
  editing, save-once and error handling MUST work as on the Spendings tab (feature 003 FR-030 to
  FR-062, as changed by feature 007), with Source in place of Merchant and income categories in
  place of spending categories.
- **FR-063**: Past sources MUST come only from the user's income entries, ranked by use, and
  accepting one MUST fill in the category and account of that source's most recent entry where
  empty.
- **FR-064**: The By category and By account cards MUST work as on the Spendings tab (features 003,
  007 and 008): only items used in the month, largest first, amount, share, total row, fixed
  colours, slice labels, every slice on the desktop. Each card's Table / Pie choice MUST be kept
  separately from the Spendings tab's.
- **FR-065**: The month line totals, footer, total card and both breakdowns MUST add up to the sum of
  the month's income entries, to the cent, and MUST update after any add, edit, delete or import
  without a full page reload.

**Phone Income tab**

- **FR-070**: The phone layout MUST offer the Spendings / Income switch near the top of the screen,
  each option a touch target of at least 44px, fitting 320 to 639px wide without sideways
  scrolling.
- **FR-071**: On Income, the phone layout MUST provide the same screens as for spendings (feature
  006, with features 007 and 008): Month and Year dropdowns, the income total, the view switch
  "Income | By category | By account", a compact list, a floating "Add income" button, full-screen
  "Add income" and "Edit income" forms, a delete confirmation sheet, and the top-6 donut.
- **FR-072**: The phone back gesture MUST keep closing the topmost screen or sheet; it MUST NOT
  switch tabs.

**Import**

- **FR-080**: The Income tab MUST offer Import CSV with the same file format, reading, review,
  fixing, sample download and all-or-nothing, retry-safe save as on Spendings (feature 004, as
  changed by features 006 and 007), on desktop and phone.
- **FR-081**: An import started on the Income tab MUST save only income entries; categories MUST be
  checked against income categories, and the sample file MUST contain income examples.
- **FR-082**: The import window MUST say whether it is importing spendings or income.

**Settings**

- **FR-090**: Settings MUST have three sections on desktop and phone: Spending categories (the
  existing Categories section, renamed), Income categories, and Accounts.
- **FR-091**: Income categories MUST list the two defaults (marked as defaults, not removable) and
  the user's own, each with a colour swatch and its entry count, with add and remove following the
  existing messages (FR-022, FR-023).

**General**

- **FR-095**: Every new control MUST be usable with the keyboard alone and labelled for screen
  readers, and the tab switch MUST tell assistive technology which tab is selected.
- **FR-096**: All server additions MUST be backward compatible: nothing published for spendings,
  categories or accounts may change or break, except the new refusal reason when removing an
  account used by income.
- **FR-097**: Every new server operation MUST be described in the published interface document with
  examples.

### Key Entities

- **Income entry**: one amount of money received. Has a date, a source, an amount, one income
  category, one account, created and last-changed times, and belongs to one user.
- **Income category**: what kind of income it is. Has a name, whether it is a default, a colour, and
  how many income entries use it (derived). Separate from spending categories.
- **Default income category list**: Salary and Investments, given to every user, fixed.
- **Account** (from feature 002, with colour from 008): shared by spendings and income entries; now
  shown with both counts.
- **Month income summary** (derived): for a year and month, the income total and entry count; for a
  month, the income total per income category and per account.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: A user with an account can record their salary on the Income tab using only the
  keyboard in under 15 seconds, with no category setup.
- **SC-002**: 100% of new and existing users see exactly two default income categories, Salary and
  Investments, on first use, with no duplicates.
- **SC-003**: In 100% of checked months, the Income tab's totals and breakdowns equal the sum of
  that month's income entries to the cent, and the Spendings tab's totals are unchanged by any
  income entry.
- **SC-004**: Switching between Spendings and Income, or to another month on Income, shows the data
  within 1 second for a user with 10,000 spendings and 1,000 income entries.
- **SC-005**: After any add, edit, delete or import of income, every figure on the Income tab
  updates within 1 second with no full page reload.
- **SC-006**: In automated tests, no user can read, change, delete or reference another user's
  income entries or income categories (0 cross-user leaks).
- **SC-007**: Every flow in this spec (tab switch, view, add, edit, delete, autofill, breakdowns,
  Settings income categories, account removal rule, import, phone screens) is covered by an
  automated end-to-end test that passes, the existing Spendings tests still pass, and the screens
  pass the automated accessibility check with no serious or critical issues.
- **SC-008**: A user can find how much they received in a given month, and from which category, in
  under 10 seconds from opening the app.

## Assumptions

- **"Same mechanics as Spendings"** means everything current on Spendings: the month line, inline
  table, autofill and keyboard entry, one-click remove on desktop and confirmed delete on phone, By
  category and By account with fixed colours and slice labels, CSV import, and the phone layout.
- **Separate income categories**: income and spending categories are different lists, so the
  Spendings dropdowns and breakdowns are not cluttered with Salary and Investments, and the reverse.
- **Shared accounts**: one list of accounts for both, since a salary is paid into the same bank
  account that spendings come out of.
- **"Source"** names who paid the income, in place of "Merchant". The design may adjust the label.
- **Entry count wording**: counts read "N entries" ("1 entry") on the Income tab.
- **Tab placement**: on the desktop the Spendings / Income tabs sit at the top of the signed-in
  screen, above the month line; the design may adjust their exact look.
- **Tab memory**: the app always opens on Spendings; the selected tab is not remembered across
  reloads.
- **Same CSV format**: the income file uses the same columns as the spendings file, the merchant
  column being read as the source; the sample file and column names are adjusted to say Source.
- **Out of scope**: net balance or savings (income minus spendings), budgets, transfers between
  accounts, recurring income, taxes and gross versus net pay, refunds, and moving an entry from
  Spendings to Income or the reverse.
- **Single currency and time zone**, as in feature 002.

## Dependencies

- **Feature 002**: user-scoped data, the default-category rule, validation, idempotent create and
  the account removal rule, all mirrored for income.
- **Feature 003**: the desktop Spendings screen, its server totals and past-merchant lookup, all
  mirrored for income.
- **Feature 004**: CSV import, reused for income.
- **Feature 006**: the phone layout, its view switch, forms and sheets, reused for income.
- **Feature 007**: the colour palette and assign-on-create rule (now also for income categories),
  slice labels, and creating categories and accounts only in Settings.
- **Feature 008**: account colours and the By account breakdown, reused on the Income tab.
