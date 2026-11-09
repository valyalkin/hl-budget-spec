# Feature Specification: Overview tab

**Feature Branch**: `011-overview-tab`

**Created**: 2026-11-08

**Status**: Draft

**Input**: User description: "I would like to add Overview tab. Design is already done in one of the tabs here: https://claude.ai/artifact/QTxGKQCY8eE5SUq7387QCZ#page-dcb81aee8d83 Use ECharts library for all charts. Overview sits right of Income. It should show by yea earned, spent, and saved (left over), then earned vs/spent chart and table by month  and spent by category chart and table, and spent by account chart and table. Refer to the overview tab design for UI (except location of the tab). Optimize for phone screen as well."

## Overview

HL Budget shows spendings and income one month at a time. This feature adds an **Overview** tab,
to the right of **Income**, that looks at a whole year at once: how much was earned, how much was
spent, how much was saved, how that went month by month, and where the money went by category
and by account.

The Overview tab is read-only. It adds no way to record, edit or delete anything; it only
summarises the spendings and income already recorded.

The look follows the **Overview** artboard of the HL-Budget design canvas
(https://claude.ai/artifact/QTxGKQCY8eE5SUq7387QCZ), with two differences: in the design the tab
sits first, before Spendings, while in the app it sits last, after Income; and the design's
"Left over" tile is called **Saved**. The screen, top to bottom:

1. A heading "Overview" with a line naming the period covered, and a year picker (previous year,
   the year, next year).
2. Three total tiles: **Earned**, **Spent** and **Saved**.
3. An **Earned vs spent** card: a month-by-month column chart, or a table by month.
4. A **Spent by category** card: a ranked bar chart, or a table.
5. A **Spent by account** card: a ranked bar chart, or a table.

On a wide screen cards 4 and 5 sit side by side; on a phone everything stacks in one column.

Nothing about the Spendings and Income tabs changes, apart from the tab switch that now also leads
to Overview.

## User Scenarios & Testing *(mandatory)*

### User Story 1 - See how my year is going (Priority: P1)

The signed-in user opens the Overview tab and sees, for the current year so far, how much they
earned, how much they spent and how much was saved, each with a short note underneath.

**Why this priority**: "Did I save money this year, and how much?" is the question the tab exists
to answer; the three tiles answer it on their own.

**Independent Test**: With income and spendings recorded through the server in several months of
the current year, open Overview and check the three tiles against the sum of the recorded entries.

**Acceptance Scenarios**:

1. **Given** a signed-in user on Spendings or Income, **When** they choose Overview, **Then** the
   Overview tab opens on the year of the month selected on that tab.
2. **Given** the current year, **When** Overview is shown, **Then** the line under the heading
   reads "January to [current month] [year] (year to date)"; for a past year it reads "January to
   December [year]".
3. **Given** the Earned tile, **When** shown, **Then** it gives the total of all income dated in
   the year and, underneath, the average per month ("avg $X a month"), where the number of months
   is 12 for a past year and the months up to and including the current month for the current
   year.
4. **Given** the Spent tile, **When** shown, **Then** it gives the total of all spendings dated in
   the year and the average per month, counted the same way.
5. **Given** the Saved tile, **When** earned is more than or equal to spent, **Then** it shows
   earned minus spent and, underneath, that amount as a share of earned ("32% of what you
   earned").
6. **Given** the Saved tile, **When** spent is more than earned, **Then** it shows the
   difference with a minus sign (for example "−$1,240") and, underneath, "Spent more than earned".
7. **Given** a year with spendings but no income, **When** the Saved tile is shown, **Then** it
   shows the negative amount and "Spent more than earned", never a share of zero.
8. **Given** a year with nothing recorded, **When** Overview is shown, **Then** all three tiles show
   $0, the averages show $0, and the Saved note reads "Nothing recorded in [year]".
9. **Given** the user opens or reloads the app, **When** it loads, **Then** it opens on the
   Spendings tab for the current month, as today.

---

### User Story 2 - Compare earned and spent month by month (Priority: P1)

Below the tiles, the Earned vs spent card shows a column chart with a pair of columns for each
month of the year, earned and spent side by side, so the user can spot the months where they spent
more than they earned. A Chart / Table switch shows the same figures as a table by month instead.

**Why this priority**: The year totals hide when money was saved or lost; the monthly view is what
makes the totals explainable.

**Independent Test**: With income and spendings in January, March and October of the current year,
check the chart has columns for those three months and none for the rest, hover a month and check
the figures; switch to Table and check each row against the recorded entries.

**Acceptance Scenarios**:

1. **Given** the Earned vs spent card in Chart view, **When** shown, **Then** it has all twelve
   months along the bottom (Jan to Dec), an earned column (blue) and a spent column (orange) for
   each month, a money scale on the side, and a legend naming Earned and Spent.
2. **Given** the current year, **When** the chart is shown, **Then** months after the current month
   have no columns.
3. **Given** the chart, **When** the user hovers, focuses or taps a month, **Then** a label shows
   the month and its earned and spent amounts to the cent.
4. **Given** the card, **When** the user switches to Table, **Then** a table lists one row per
   month (January to December for a past year; January to the current month for the current
   year) with columns Month, Earned, Spent and Net, where Net is earned minus spent, shown with a
   minus sign when negative; a month with nothing recorded shows $0.
5. **Given** the card's Chart / Table choice, **When** the user changes year, leaves the tab and
   comes back, or reloads the app, **Then** the choice is kept.
6. **Given** a year with nothing recorded, **When** the card is shown, **Then** it shows "Nothing
   recorded in [year]." instead of an empty chart or table.

---

### User Story 3 - See where the year's money went (Priority: P2)

The Spent by category card shows the year's total spent and a ranked bar chart of every category
the user spent in that year, largest first. The Spent by account card shows the same for accounts.
Each card has its own Chart / Table switch; the table lists each category (or account) with the
amount spent and its share of the year's spending.

**Why this priority**: Very useful for deciding what to cut, but the tab already answers its main
question without it, and the monthly Spendings tab already has per-month breakdowns.

**Independent Test**: With a year of spendings in five categories across two accounts, check Spent
by category lists the five categories largest first, their amounts add up to the Spent tile, and
switching to Table shows the same amounts and shares; check the same for Spent by account.

**Acceptance Scenarios**:

1. **Given** the Spent by category card, **When** shown, **Then** its header shows the year's total
   spent (equal to the Spent tile) with "total spent in [year]".
2. **Given** Chart view, **When** shown, **Then** each category with spendings in the year has one
   horizontal bar, largest at the top, its name beside it and a money scale along the bottom;
   categories with no spendings in the year are not listed.
3. **Given** the chart, **When** the user hovers, focuses or taps a bar, **Then** a label shows the
   category, its amount to the cent and its share of the year's spending ("$7,680.00 · 24% of
   spent").
4. **Given** Table view, **When** shown, **Then** it lists the same categories in the same order
   with columns Category, Spent and Share, and the amounts add up to the year's total spent.
5. **Given** the Spent by account card, **When** shown, **Then** its header reads "Where the [total
   spent] came from", and its chart and table behave as in scenarios 2 to 4, with accounts in
   place of categories.
6. **Given** a user with many categories, **When** the chart is shown, **Then** it grows taller so
   every category name stays readable; no category is dropped or merged into "Other".
7. **Given** each card's Chart / Table choice, **When** the user changes year, leaves the tab or
   reloads, **Then** each card keeps its own choice.
8. **Given** a year with no spendings, **When** the cards are shown, **Then** each shows "No
   spendings in [year]." instead of an empty chart or table.
9. **Given** income, **When** either card is shown, **Then** income is never counted: both cards
   are about money spent only.

---

### User Story 4 - Look at another year (Priority: P2)

With the arrows beside the year, the user steps back to earlier years and forward again. Every
tile, chart and table updates to the chosen year.

**Why this priority**: Comparing with last year is a natural next question, but the current year
alone is already useful.

**Independent Test**: With entries in the previous and current year, open Overview, press the
previous-year arrow, check every figure matches the previous year, press next and check the current
year is back.

**Acceptance Scenarios**:

1. **Given** Overview, **When** the user presses the previous-year arrow, **Then** the year goes
   back by one and every tile, chart, table and the period line show that year.
2. **Given** the current year, **When** Overview is shown, **Then** the next-year arrow is shown
   disabled: the user cannot move into a future year.
3. **Given** the earliest year that has any spending or income, **When** it is shown, **Then** the
   previous-year arrow is disabled; a user with nothing recorded can see only the current year.
4. **Given** a past year chosen on Overview, **When** the user switches to Spendings or Income,
   **Then** those tabs show the month they showed before; Overview's year does not move them.

---

### User Story 5 - Use the Overview on a phone (Priority: P2)

On a phone the Overview tab is reachable from the same tab switch as Spendings and Income and lays
out in one column: the year picker, the three tiles stacked, then each card full width. Charts fit
the screen width, tapping shows the same labels as hovering, and tables fit or scroll inside their
card.

**Why this priority**: The user checks their budget on the phone as often as on the desktop.

**Independent Test**: On a phone-sized screen (360 to 430 pixels wide), open Overview and check
nothing makes the page scroll sideways, every control can be tapped, all four tiles' and cards'
figures can be read, and tapping a column or bar shows its label.

**Acceptance Scenarios**:

1. **Given** the phone layout (below 640 pixels wide), **When** the user opens the tab switch,
   **Then** it offers Spendings, Income and Overview, in that order.
2. **Given** Overview on a phone, **When** shown, **Then** the year picker, the three tiles and the
   three cards are in a single column, and the page never scrolls sideways.
3. **Given** the Earned vs spent chart on a phone, **When** shown, **Then** all twelve months fit
   the width with readable labels (shortened if needed) and no sideways scrolling inside the chart.
4. **Given** any chart on a phone, **When** the user taps a column or bar, **Then** the same label
   as on the desktop appears, and tapping elsewhere hides it.
5. **Given** a table on a phone, **When** it is wider than the screen, **Then** only the table
   scrolls sideways, inside its card.
6. **Given** any button on the Overview tab on a phone (year arrows, Chart / Table switches, tab
   switch), **When** measured, **Then** its touch target is at least 44 by 44 pixels.

---

### Edge Cases

- **Year with income but no spendings**: Spent is $0, Saved equals Earned at 100%; the two
  breakdown cards show "No spendings in [year]."; the monthly chart shows only earned columns.
- **A month with only one side**: the month shows one column; its table row shows $0 on the other
  side.
- **Entries recorded or changed on another tab**: when the user comes back to Overview, it shows
  figures that include every change made since, without a manual reload.
- **Removal with Undo pending (feature 010)**: an entry being undone is not yet deleted; Overview
  counts what is stored when it loads.
- **Large amounts**: totals of hundreds of thousands of dollars still fit the tiles and the chart
  scales on both desktop and phone.
- **Long category or account names**: they are shown in full in the table; in the chart they may be
  shortened with the full name in the hover/tap label.
- **The data fails to load**: the tab shows a short message with a Try again button, like the other
  tabs, and no partial figures that would be wrong.
- **Year change near midnight on 31 December**: "current year" and "current month" are those of the
  user's local date, as on the other tabs.

## Requirements *(mandatory)*

### Functional Requirements

**Tab and navigation**

- **FR-001**: The app MUST show an Overview tab to the right of Income, on the desktop and in the
  phone tab switch.
- **FR-002**: Opening Overview from Spendings or Income MUST show the year of the month selected
  there; changing the year on Overview MUST NOT change the month shown on Spendings or Income.
- **FR-003**: The app MUST still open on Spendings for the current month when loaded or reloaded.
- **FR-004**: The Overview tab MUST be read-only: no add, edit, delete or import controls.

**Year picker and period**

- **FR-005**: Overview MUST show one calendar year at a time, defaulting per FR-002, with previous-
  and next-year arrows.
- **FR-006**: The next-year arrow MUST be disabled on the current year; the previous-year arrow MUST
  be disabled on the earliest year with any spending or income (the current year when there is
  none).
- **FR-007**: A line under the heading MUST name the period: "January to December [year]" for a
  past year, "January to [current month] [year] (year to date)" for the current year.

**Totals**

- **FR-008**: Earned MUST be the sum of all income dated in the chosen year; Spent MUST be the sum
  of all spendings dated in the chosen year; Saved MUST be Earned minus Spent.
- **FR-009**: Earned and Spent MUST each show the average per month, dividing by 12 for a past year
  and by the number of months up to and including the current month for the current year.
- **FR-010**: Saved MUST show its share of Earned when Earned is above zero and Saved is
  zero or more; "Spent more than earned" when it is negative; "Nothing recorded in [year]" when the
  year has no entries.
- **FR-011**: Negative amounts MUST be shown with a minus sign in front of the dollar sign.

**Earned vs spent**

- **FR-012**: The card MUST offer Chart and Table views, Chart by default.
- **FR-013**: The chart MUST show, for each of the twelve months, an earned column and a spent column
  side by side, in the Earned and Spent colours of the tiles, with a legend, and no columns for
  future months.
- **FR-014**: Hovering, focusing or tapping a month MUST show its name and earned and spent amounts
  to the cent.
- **FR-015**: The table MUST list each month of the period (FR-007) with Earned, Spent and Net
  (earned minus spent).

**Spent by category and Spent by account**

- **FR-016**: Each card MUST offer Chart and Table views, Chart by default, chosen separately per
  card.
- **FR-017**: The chart MUST be a horizontal bar per category (or account) with spendings in the
  year, sorted largest first, with every name readable; nothing is grouped into "Other".
- **FR-018**: Hovering, focusing or tapping a bar MUST show the name, amount to the cent and share
  of the year's spending.
- **FR-019**: The table MUST list the same rows in the same order with the amount and its share of
  the year's spending; the amounts MUST add up to the Spent tile.
- **FR-020**: The Spent by category header MUST show the year's total spent; the Spent by account
  header MUST read "Where the [total spent] came from".
- **FR-021**: Both cards MUST count spendings only, never income.

**Display rules**

- **FR-022**: Amounts in tiles and tables MUST be shown in Canadian dollars rounded to whole
  dollars with thousands separators (for example "$12,480"); hover/tap labels MUST show cents.
  Shares MUST be whole percentages, with one decimal below 1% (for example "0.4%").
- **FR-023**: Each card's Chart / Table choice MUST be remembered on the device across year
  changes, tab switches and reloads.
- **FR-024**: Every chart MUST have a text alternative for screen readers that states what it shows
  and its key figures (for example the year's totals, or the largest category and its amount).
- **FR-025**: Empty states MUST replace empty charts and tables: "Nothing recorded in [year]." for
  Earned vs spent and "No spendings in [year]." for the two breakdown cards.
- **FR-026**: When Overview's data cannot be loaded, the tab MUST show a short error message with a
  Try again button and no figures.
- **FR-027**: Overview MUST reflect every spending and income recorded, changed or deleted on other
  tabs whenever the user opens it.

**Phone**

- **FR-028**: Below 640 pixels wide the tab MUST lay out in one column with no sideways page scroll;
  the charts MUST fit the screen width; wide tables MUST scroll inside their card.
- **FR-029**: All Overview controls on a phone MUST have touch targets of at least 44 by 44 pixels,
  and tapping a chart MUST show the same labels as hovering on the desktop.

### Key Entities

- **Year summary**: for one user and one calendar year, the total earned, total spent and left
  over. Derived from existing income and spending entries; nothing new is stored.
- **Month summary**: for each month of the year, the total earned and total spent. Derived.
- **Category spending for the year**: for each spending category used in the year, the total spent.
  Derived.
- **Account spending for the year**: for each account used by spendings in the year, the total
  spent. Derived.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: From the Spendings tab, the user can find out how much they have saved this year
  in under 5 seconds (one tap or click).
- **SC-002**: For any year, the Earned, Spent and Saved tiles, the monthly table and both
  breakdown tables match the sum of the recorded entries to the dollar, and the hover labels to the
  cent.
- **SC-003**: Overview shows a full year with up to 3,000 entries in under 2 seconds, and switching
  year updates every figure in under 1 second.
- **SC-004**: On phone screens from 360 to 430 pixels wide, no part of the Overview tab makes the
  page scroll sideways, every figure can be read without zooming, and every control is at least 44
  by 44 pixels.
- **SC-005**: The automated accessibility check reports no serious or critical issues on the
  Overview tab, on desktop and phone, in both Chart and Table views.

## Assumptions

- **Design source**: the Overview artboard of the HL-Budget design canvas is the reference for
  layout, wording, colours (Earned blue, Spent orange) and the Chart / Table switches; only the
  tab's position (after Income, not first) and the third tile's name (Saved, not Left over)
  differ.
- **Charting library**: by the user's choice, the Overview charts are drawn with ECharts. This
  applies to the Overview tab only; the existing donuts on Spendings and Income are not changed by
  this feature.
- **Single colour for breakdown bars**: as in the design, the category and account bars all use the
  Spent colour rather than each category's or account's own colour (feature 007), since every bar
  is "money spent"; the tables likewise show no colour swatches.
- **Calendar years only**: years run January to December; no custom ranges, fiscal years or
  rolling 12 months.
- **Same currency**: everything is in CAD, as in the rest of the app; no conversion.
- **Transfers are not modelled**: Saved is simply income minus spendings; there is no notion of
  savings accounts or transfers between accounts.
- **Figures are computed where the data lives**: monthly totals for spendings and income already
  exist; year-level totals by category and by account may need to be added. The plan decides how.
- **Desktop and phone breakpoint**: the phone layout applies below 640 pixels wide, as in feature
  006.
- **Earliest year**: found from the user's earliest dated spending or income.
