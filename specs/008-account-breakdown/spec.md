# Feature Specification: Spending by account

**Feature Branch**: `008-account-breakdown`

**Created**: 2026-11-01

**Status**: Draft

**Input**: User description: "New feature - aggregation by acccount, same as for by category."

## Overview

The Spendings screen shows where the month's money went by category (feature 003 on desktop,
feature 006 on the phone, with fixed colours and slice labels from feature 007). This feature adds
the same breakdown by account: for the selected month, how much was spent from each account
("Amex", "Debit card", "Cash"), as a table or a donut chart, on desktop and phone.

It works like By category in every respect unless this spec says otherwise:

- Only accounts with spending in the selected month are shown, largest first, with amount, share
  and a total row.
- Each account has its own fixed colour, given when the account is created and kept for its
  lifetime, used by the donut, legend and table swatches.
- Hovering, focusing or tapping a slice names the account with its amount and share.
- The desktop donut shows every account; the phone donut shows the top 6 plus "Everything else".

Nothing about By category changes.

## User Scenarios & Testing *(mandatory)*

### User Story 1 - See the month by account on the desktop (Priority: P1)

The user opens the Spendings screen on the desktop and sees, next to By category, the month's
spending by account, as a table or a donut chart, chosen with a Table / Pie toggle. This answers
"which card or account did the money go out of?", for example to check a credit card bill.

**Why this priority**: This is the feature. The desktop is where the user reviews a month.

**Independent Test**: With October spendings on three accounts out of four, select October and
check the By account table lists those three (and not the unused one) with amount, share and a
total row equal to the month total; switch to Pie and check three slices with a legend.

**Acceptance Scenarios**:

1. **Given** By account in Table view, **When** a month with spendings is selected, **Then** each
   account used that month is listed with a colour swatch, its amount and its share of the month's
   total (whole-number percentage), largest first, followed by a total row equal to the month
   total.
2. **Given** By account in Pie view, **When** a month with spendings on three accounts is
   selected, **Then** a donut shows three slices, one per account, largest first, each in its
   account's colour, with a legend giving each slice's name, amount and share. There is no
   "Everything else" slice.
3. **Given** accounts with no spending in the selected month, **When** either view is shown,
   **Then** those accounts do not appear.
4. **Given** a month with no spendings, **When** By account is shown, **Then** it shows a short
   empty message instead of a table or chart.
5. **Given** the user chose Pie (or Table) for By account, **When** they change month, **Then**
   the chosen view is kept.
6. **Given** the user adds, edits or deletes a spending, or adds a CSV import, **When** it is
   saved, **Then** By account shows the updated figures without a page reload, as By category
   does.
7. **Given** a spending moved from "Amex" to "Cash" by editing it, **When** it is saved, **Then**
   its amount leaves Amex's total and joins Cash's total.

---

### User Story 2 - Each account has its own colour and named slices (Priority: P1)

Each account is always drawn in the same colour, in every month, so the user learns them. Pointing
at, focusing or tapping a slice shows the account name, amount and share at once.

**Why this priority**: Without fixed colours and labels the donut is hard to read, which was the
bug fixed for categories in feature 007. The account donut must not repeat it.

**Independent Test**: Create four accounts, record spendings on all four in two months, and check
the four slices have four different visible colours, matching their legend and table swatches,
the same in both months. Hover each slice and check the label.

**Acceptance Scenarios**:

1. **Given** a month with spending on four accounts, **When** the user opens Pie view, **Then**
   all four slices are visibly coloured, each account in a different colour, matching its legend
   swatch and its table swatch.
2. **Given** spending on "Amex" in October and November, **When** the user views each month,
   **Then** Amex has the same colour in both, whatever its rank.
3. **Given** the user creates an account "Savings" in Settings, **When** they record spending on it
   and open Pie view, **Then** Savings has a colour not used by any of their other accounts, kept
   in every month and after reloading.
4. **Given** Pie view on the desktop, **When** the user hovers a slice, **Then** straight away a
   label shows the account name, amount and share, for example "Amex · $1,240.00 · 38%", and the
   slice is visibly emphasised; leaving the chart hides it.
5. **Given** a keyboard user, **When** they move focus onto the slices, **Then** each focused slice
   shows the same label and screen readers announce the same text.

---

### User Story 3 - See the month by account on the phone (Priority: P2)

On the phone layout (below 640px wide) the user switches to By account, alongside Spendings and
By category, and gets the same "Table | Pie" toggle as By category.

**Why this priority**: Useful on the go, but the desktop is where months are reviewed.

**Independent Test**: On a phone-width screen with October spendings on 8 accounts, switch to By
account, check the table lists all 8 with amounts, shares and a total equal to the month total;
switch to Pie and check 6 named slices plus "Everything else" whose amount equals the other 2
accounts.

**Acceptance Scenarios**:

1. **Given** the phone Spendings screen, **When** the user taps "By account", **Then** the list is
   replaced by the account breakdown with a "Table | Pie" toggle; the month dropdowns and total
   stay.
2. **Given** Table view, **When** it shows, **Then** it lists every account with spending in the
   month, highest first, with swatch, amount and share, and a final "Total" row with the month
   total and 100%.
3. **Given** Pie view with more than 6 accounts, **When** it shows, **Then** the donut has one
   slice for each of the 6 highest accounts, one "Everything else" slice for the rest, the month
   total in the middle, and a legend with each slice's name, colour, amount and share.
4. **Given** Pie view with 6 or fewer accounts, **When** it shows, **Then** every account has its
   own slice and there is no "Everything else".
5. **Given** Pie view, **When** the user taps a slice, **Then** its label is shown; tapping
   elsewhere hides it.
6. **Given** a month with no spendings, **When** By account is shown, **Then** it shows "Nothing to
   break down in [Month Year]."
7. **Given** By account and Pie (or Table) chosen, **When** the user changes the month, **Then**
   the same view stays selected.

---

### Edge Cases

- **A single account in the month**: the table has one row at 100% and the donut is one full ring
  in that account's colour; hover still names it.
- **User with no accounts**: they have no spendings, so By account shows the empty message.
- **Renamed account**: the breakdown shows the new name, keeps the colour, and its spendings stay
  under it, including past months.
- **Removed account**: an account can only be removed when no spending uses it (feature 002), so
  it never appears in any breakdown; its colour becomes free for the next new account.
- **More accounts than spare colours**: once every colour is in use, a new account reuses the
  colour used by the fewest of the user's accounts; the slice gap, legend and hover label still
  tell slices apart.
- **Accounts that existed before this feature**: each is given a colour once, by the same rule as
  new accounts.
- **Shares that don't add up to 100 after rounding**: the total row always shows 100%; individual
  shares are rounded the same way as in By category.
- **Very small shares and long names**: as in By category: a tiny account still gets a hoverable
  slice and a legend entry; long names wrap in the label and stay inside the card.

## Requirements *(mandatory)*

### Functional Requirements

**Breakdown**

- **FR-001**: The Spendings screen MUST show the selected month's spending by account on desktop
  and phone layouts, alongside the existing By category breakdown, which MUST stay unchanged.
- **FR-002**: The By account breakdown MUST offer a Table / Pie toggle of its own; its chosen view
  MUST be kept when the month changes.
- **FR-003**: Table view MUST list every account with spending in the selected month, and only
  those, with a colour swatch, its amount and its share of the month's total (whole-number
  percentage), largest first, plus a total row with the month total.
- **FR-004**: On the desktop, Pie view MUST show a donut with one slice for every account with
  spending in the month, with no cap and no "Everything else" grouping, and a legend giving each
  slice's name, amount and share, largest first.
- **FR-005**: On the phone, Pie view MUST show a donut with one slice for each of the 6 highest
  accounts and, when there are more, one "Everything else" slice for the rest, with the month
  total in the middle and a legend giving each slice's name, colour, amount and share.
- **FR-006**: A month with no spendings MUST show a short empty message instead of the table or
  chart ("Nothing to break down in [Month Year]." on the phone).
- **FR-007**: The sum of the account amounts MUST equal the month total shown elsewhere on the
  screen, to the cent.
- **FR-008**: After any add, edit or delete of a spending, or an import being added, the By
  account breakdown MUST update together with the rest of the screen, without a full page reload.

**Desktop placement**

- **FR-010**: On the desktop, By account MUST be its own card, placed directly below the By
  category card in the same column of summary cards, with the same look; when the window is too
  narrow and the summary cards move below the table (feature 003 FR-024), it moves with them.

**Phone placement**

- **FR-015**: On the phone, the view switch MUST become three options, "Spendings | By category |
  By account", with Spendings selected on opening; each option MUST remain a touch target of at
  least 44px and fit the screen width without scrolling sideways.

**Account colours**

- **FR-020**: Each account MUST have a colour, given when it is created and kept for its lifetime,
  including after renaming.
- **FR-021**: A new account's colour MUST differ from the user's other accounts' colours while the
  palette has unused colours; after that, the colour used by the fewest of the user's accounts is
  reused.
- **FR-022**: Accounts that exist before this feature MUST each be given a colour once, by the
  FR-021 rule.
- **FR-023**: Account colours MUST come from the same palette as category colours and be clearly
  visible against the card background (at least 3:1 contrast).
- **FR-024**: The pie slices, the pie legend and the table swatches MUST all show each account in
  its own colour, on desktop and phone.

**Slice labels and accessibility**

- **FR-030**: Hovering a slice MUST show, without a noticeable delay, a visible label with the
  account name, amount and share, and visibly emphasise the slice; moving to another slice MUST
  update it and leaving the chart MUST hide it.
- **FR-031**: Slices MUST be reachable by keyboard; a focused slice MUST show the same label and
  expose the same text to assistive technology.
- **FR-032**: On the phone, tapping a slice MUST show the same label and tapping elsewhere MUST
  hide it; no information MUST be available by hover alone.
- **FR-033**: The chart's values MUST also be available as text (the legend) to screen readers,
  and every new control MUST be usable with the keyboard alone and labelled.

**Server**

- **FR-040**: Per-account totals for a month MUST be available from the server for the signed-in
  user only, counting only that user's spendings; any server addition MUST be backward compatible.

### Key Entities

- **Account**: as in feature 002, plus a colour assigned when it is created.
- **Account total** (derived): for a month, the total amount and share of the month spent from one
  account. Shown in the By account table and donut.
- **Category colour palette**: the palette from feature 007, now also used for account colours.
  Account and category colours are assigned independently, so an account may share a colour with
  a category.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: In 100% of checked months, the By account amounts add up to the month total to the
  cent, and each account's amount matches the sum of that month's spendings on it.
- **SC-002**: Switching to another month shows its account breakdown within 1 second for a user
  with 10,000 spendings in total, as for By category.
- **SC-003**: After any add, edit, delete or import, the By account breakdown shows the updated
  figures within 1 second, with no full page reload.
- **SC-004**: 100% of donut slices are visibly coloured and match their legend and table swatches;
  an account has the same colour in 100% of months.
- **SC-005**: A user's first accounts (up to the palette size) each get a colour not used by any
  of their other accounts.
- **SC-006**: Hovering, focusing or tapping any slice shows its account name within half a second,
  for 100% of slices.
- **SC-007**: The user can find how much they spent from a given account in a month in under 10
  seconds from opening the Spendings screen.
- **SC-008**: Every flow in this spec (desktop table and pie, phone table and pie, slice labels,
  empty month) is covered by an automated end-to-end test that passes, and the screens pass the
  automated accessibility check with no serious or critical issues.

## Assumptions

- "Same as for by category" means the full current By category behaviour, including feature 007's
  fixed colours and slice labels and feature 006's phone top-6 donut.
- Desktop placement: a separate By account card below the By category card, so both breakdowns are
  visible at once without extra clicks. The design may adjust its exact styling.
- Account colours reuse the category palette; colours are assigned automatically and the user
  cannot choose them in this feature.
- The breakdown is for the selected month only, like By category; ranges, years and comparisons
  are out of scope.
- No filtering: clicking a slice or row does not filter the spendings list.
- Amounts are in the app's single currency (feature 002), shown with two decimals.
- Settings needs no visible change; it may show each account's colour next to its name, but that
  is not required.

## Dependencies

- **Feature 002**: accounts on the server, scoped to the user, and the rule that an account in use
  cannot be removed.
- **Feature 003**: the Spendings screen, the summary cards and the By category card this feature
  mirrors; the server's month totals per category, which By account mirrors for accounts.
- **Feature 006**: the phone Spendings screen, its view switch and the top-6 donut.
- **Feature 007**: the colour palette, the assign-on-create colour rule and the slice labels,
  now applied to accounts too.
