# Feature Specification: Fixed category colours, a working pie chart, and categories and accounts managed only in Settings

**Feature Branch**: `007-fix-pie-csv-categories`

**Created**: 2026-10-31

**Status**: Draft

**Input**: User description: "Bug fixing: Pie chart doesn't work correctly. Correct behavior is that each category is getting assigned a color, but now I see only rent in color and some other random categories. Also on hover it must say the category as well. That is one. I would like NOT to create a new category when CSV is imported. It shoudl highlight it as invalid category." Follow-up: "Also prevent new accounts to be created. Don't create acounts or categories outside of settings, even on the spendings screens. Colors should be the same for default categories and new colors assigned to new categoris."

## Overview

Three changes to behaviour shipped in features 002 (Spending Tracking), 003 (Spending Screens),
004 (CSV Import) and 006 (Mobile View):

1. **Each category has its own fixed colour.** Each of the 17 default categories always has the
   same colour, for every user and every month. A category the user creates in Settings is given
   a new colour when it is created and keeps it. The pie chart and its legend use these colours.
   Today only some slices are coloured (the largest, usually Rent, and a few others) and colours
   follow slice size, so a category's colour changes from month to month.
2. **Hovering a slice names the category.** Pointing at, focusing or tapping a slice shows its
   category name, amount and share straight away.
3. **Categories and accounts are created only in Settings.** Adding or editing a spending (on
   desktop or phone) and importing a CSV file only accept existing categories and accounts. An
   unknown name is shown as an invalid field and is never created.

This replaces these parts of earlier specs: feature 003 FR-072's rank-based colours and the
"creates a missing category or account" rule (scenario 6, FR-035's second sentence, "no partly
created items" edge case); feature 004 FR-032's creation of names, scenario 10 and the "Data changed
elsewhere" edge case; feature 006 FR-048 and FR-096's creation of names; and feature 002's
creating a missing category or account together with a spending.

## Clarifications

### Session 2026-10-31

- Q: Should unknown accounts in a CSV still be created? → A: No. Like categories, they are
  flagged as invalid and never created.
- Q: Should manual entry on the Spendings screen still create a typed category or account? → A:
  No. Categories and accounts are created only in Settings, on desktop and phone.
- Q: Should slice colours follow size order or belong to each category? → A: Each category has a
  fixed colour: the defaults always have the same colours, and each new category is assigned a
  new colour.

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Every category has its own colour in the pie (Priority: P1)

The user opens the Spendings screen, switches By category to Pie and sees a donut where every
category is drawn in its own visible colour, matching the legend. Rent is the same colour in every
month, so the user learns the colours and can read the chart at a glance.

**Why this priority**: The chart is the main way to see where money went. With most slices
uncoloured it is unreadable, so this fixes a broken feature.

**Independent Test**: Pick a month with spending in all 17 default categories and check that all
17 slices have different, visible colours that match their legend swatches. Switch to another
month and check that each category keeps the same colour. Repeat as a second user and check that
the defaults have the same colours.

**Acceptance Scenarios**:

1. **Given** a month with spending in 8 categories, **When** the user opens Pie view, **Then**
   all 8 slices are drawn in visible colours, each category in its own colour, and each legend
   swatch shows the same colour as its slice.
2. **Given** spending in Groceries in October and in November, **When** the user views each
   month, **Then** Groceries has the same colour in both, whatever its rank.
3. **Given** two different users, **When** each views a month with Rent spending, **Then** Rent
   has the same colour for both.
4. **Given** a month with spending in all 17 default categories, **When** the user opens Pie
   view, **Then** all 17 colours are different from each other.
5. **Given** the user creates a category "Coffee" in Settings, **When** they record Coffee
   spending and open Pie view, **Then** Coffee has a colour not used by any of their other
   categories, and it keeps that colour in every month and after reloading.
6. **Given** the phone layout (below 640px wide), **When** the user opens By category as a
   donut, **Then** every slice uses its category's colour and matches its legend entry.
7. **Given** By category in Table view, **When** the user looks at the rows, **Then** each row
   shows a small swatch in its category's colour, so table and pie use the same colours.

---

### User Story 2 - Hovering a slice names the category (Priority: P1)

The user points at a slice (or moves keyboard focus to it, or taps it on a phone) and immediately
sees the category name, its amount and its share of the month, without matching colours against
the legend.

**Why this priority**: The user asked for it directly. Colour alone cannot identify a slice
reliably, especially for colour-blind users.

**Independent Test**: Open Pie view, move the pointer over each slice in turn and check that a
visible label appears at once with that slice's category, amount and share, and disappears when
the pointer leaves the chart.

**Acceptance Scenarios**:

1. **Given** Pie view on the desktop, **When** the user hovers a slice, **Then** straight away (no
   long browser delay) a visible label shows the category name, its amount and its share of the
   month, for example "Rent · $1,850.00 · 42%".
2. **Given** a slice is hovered, **When** the pointer moves to another slice, **Then** the label
   changes to that slice's category; **When** the pointer leaves the chart, **Then** the label
   goes away.
3. **Given** a hovered slice, **When** the label is shown, **Then** the slice is also visibly
   emphasised (for example raised, outlined or the others dimmed).
4. **Given** a keyboard user, **When** they move focus onto the chart's slices, **Then** each
   focused slice shows the same label and screen readers announce the same text.
5. **Given** the phone layout, **When** the user taps a slice, **Then** the same label is shown;
   tapping outside the chart hides it.

---

### User Story 3 - Spendings use only existing categories and accounts (Priority: P1)

When adding or editing a spending on the Spendings screen (desktop entry row or phone Add / Edit
screen), the user picks a category and an account from their existing ones. Typing a name that
does not exist marks the field invalid with a short reason and the spending cannot be saved until
an existing name is chosen. New categories and accounts are made in Settings.

**Why this priority**: The user asked for it directly. Creating items from a typing slip fills the
lists with typos and duplicates and splits the breakdown across near-identical names.

**Independent Test**: In the desktop entry row, type "Groceriess" as the category and try to save:
the field is highlighted with "Not one of your categories" and nothing is saved. Change it to
"groceries": it shows as "Groceries" and the spending saves. Do the same for an account and on
the phone Add screen. Check that Settings lists the same categories and accounts as before.

**Acceptance Scenarios**:

1. **Given** the entry row or phone Add / Edit screen, **When** the user types into the category
   or account field, **Then** the suggestions offer only existing categories or accounts.
2. **Given** the user types a category that matches an existing one in any letter case, **When**
   they leave the field or save, **Then** it is shown with the existing name and is valid.
3. **Given** the user types a category that matches no existing category, **When** they leave the
   field or try to save, **Then** the field is highlighted as invalid with "Not one of your
   categories", marked invalid for assistive technology, and the spending is not saved; nothing
   is created.
4. **Given** the same for an account, **Then** the reason is "Not one of your accounts".
5. **Given** an invalid category or account, **When** the user picks or types an existing one,
   **Then** the field becomes valid and the spending can be saved with Tab / Enter or the save
   button as before.
6. **Given** a user with no accounts yet, **When** they start adding a spending, **Then** they see
   a short message that they need to add an account in Settings first, with a way to open
   Settings.
7. **Given** editing an existing spending, **When** the user changes its category to an unknown
   name, **Then** the same rule applies and the change is not saved.

---

### User Story 4 - An unknown category or account in a CSV is flagged, not created (Priority: P1)

The user imports a CSV file in which some rows use a category or account that does not exist. In
the review, those fields are highlighted as invalid with a short reason, the add button stays
greyed out, and the user picks an existing name for each such row or removes the row. Adding the
import never creates a category or an account.

**Why this priority**: The user asked for it directly, for the same reason as Story 3.

**Independent Test**: Import a file with four rows: one with "groceries" (existing, other case),
one with "Groceriess", one with an unknown account "Visa Gold" and one with no category. Check
that the first is valid and shown as "Groceries", the other three are highlighted, and "Add
spendings" is greyed out. Fix them with existing names, add, and check that Settings shows the
same categories and accounts as before.

**Acceptance Scenarios**:

1. **Given** a row whose category or account matches an existing one in any letter case, **When**
   the review is shown, **Then** the field shows the existing name and is valid (as in feature
   004 FR-014).
2. **Given** a row whose category matches no existing category, **When** the review is shown,
   **Then** the field keeps the name from the file, is highlighted as invalid, is marked invalid
   for assistive technology, gives the reason "Not one of your categories", and the row counts
   toward "N rows need attention". An unknown account behaves the same with "Not one of your
   accounts".
3. **Given** an invalid field, **When** the user types into it, **Then** suggestions offer only
   existing names, and the field becomes valid as soon as its value matches one.
4. **Given** any row has an unknown category or account, **When** the user looks at the footer,
   **Then** the add button is greyed out and labelled "Add spendings"; fixing or removing the row
   re-enables it if nothing else is invalid.
5. **Given** every row is valid, **When** the user adds the import, **Then** all spendings are
   saved and the categories and accounts in Settings are unchanged.
6. **Given** the phone layout, **When** the review list shows a row with an unknown category or
   account, **Then** the row is marked as needing attention, and the Fix row screen highlights
   the field with the same reason and offers only existing names.

---

### Edge Cases

- **Name removed meanwhile**: if a category or account is removed (in Settings or another tab)
  after the user chose it but before saving, the save or import is refused with a short message,
  nothing is saved, the entry or review stays open with the user's input, and the field is
  highlighted with "Not one of your categories" or "Not one of your accounts".
- **Name added meanwhile**: a category or account created in Settings in another tab becomes
  valid the next time names are checked (at the latest on save); no reload is needed.
- **Spaces and letter case**: leading or trailing spaces are ignored and letter case does not
  matter when matching; "  groceries " matches "Groceries".
- **Empty category or account**: still invalid as before, with its existing message.
- **Many rows with the same unknown name**: each such row is highlighted; there is no bulk
  "replace all" in this fix.
- **Accepting a past merchant**: autofilling category and account from a past merchant (feature
  003) only fills names that still exist; a removed one is left as an invalid field.
- **Categories and accounts already created by earlier imports or typing**: they stay as normal
  user items; this fix does not remove or merge them. Custom categories that exist before this
  fix are each given a colour once, by the same rule as new ones.
- **More custom categories than spare colours**: once every colour in the palette is in use, a
  new category reuses a colour, choosing the one used by the fewest of the user's categories.
  Slices stay separated by a thin gap, and hover labels and the legend still identify them.
- **Removing a custom category**: its colour becomes free for the next new category; it does not
  change any other category's colour.
- **Renaming a custom category**: keeps its colour.
- **Very small categories in the pie**: a category with a very small share still gets a slice in
  its colour and can be hovered; the legend always lists it.
- **Long category names**: the hover label shows the full name, wrapping if needed, and stays
  inside the chart card.

## Requirements *(mandatory)*

### Functional Requirements

**Category colours**

- **FR-001**: Each of the 17 default categories MUST have a fixed colour, the same for every user
  and every month, and all 17 MUST be different.
- **FR-002**: A category the user creates MUST be given a colour when it is created and keep it
  for its lifetime, including after renaming. The colour MUST differ from all of the user's other
  categories' colours while the palette has unused colours; after that, the least-used colour is
  reused.
- **FR-003**: Every colour MUST be clearly visible against the card background (at least 3:1
  contrast).
- **FR-004**: The pie slices, the pie legend and the By category table MUST all show each
  category in its own colour, on both desktop and phone layouts.
- **FR-005**: The user's custom categories that exist before this change MUST each be given a
  colour once, by the FR-002 rule.

**Pie chart labels**

- **FR-010**: Every slice MUST be drawn in its category's colour and be visibly filled.
- **FR-011**: Hovering a slice with a pointer MUST show, without a noticeable delay, a visible
  label with the category name, amount and share of the month; moving to another slice MUST
  update it, and leaving the chart MUST hide it.
- **FR-012**: The hovered or focused slice MUST be visibly emphasised while its label is shown.
- **FR-013**: Slices MUST be reachable by keyboard; a focused slice MUST show the same label and
  expose the same text to assistive technology.
- **FR-014**: On the phone layout, tapping a slice MUST show the same label and tapping elsewhere
  MUST hide it. No information MUST be available by hover alone.

**Creating categories and accounts only in Settings**

- **FR-020**: Categories and accounts MUST be creatable only from Settings (desktop and phone).
  No other action may create one.
- **FR-021**: When adding or editing a spending (desktop entry row, phone Add / Edit screen), the
  category and account MUST each match an existing item, ignoring letter case and leading or
  trailing spaces; a match MUST be shown with the existing name.
- **FR-022**: A category or account that is not empty and matches no existing item MUST be
  highlighted as invalid, marked invalid for assistive technology, and show "Not one of your
  categories" or "Not one of your accounts"; the spending MUST NOT be saved while it is invalid.
- **FR-023**: Suggestions for category and account fields, everywhere a spending is entered or
  fixed, MUST offer only existing items.
- **FR-024**: A user with no accounts who starts adding a spending MUST see a short message
  pointing them to Settings to add one, with a way to open Settings.
- **FR-025**: In the CSV review (desktop table and phone Fix row screen), a category or account
  that matches no existing item MUST be treated as invalid per FR-022, MUST keep the text from
  the file, MUST count toward the "rows need attention" badge, and MUST keep the add button
  greyed out (feature 004 FR-022, FR-031).
- **FR-026**: Adding an import MUST NOT create any category or account. Saving a spending MUST NOT
  create any category or account.
- **FR-027**: If a category or account no longer exists when a save or import is attempted, the
  whole save or import MUST be refused, nothing saved, the user's input kept, and the affected
  fields highlighted per FR-022.
- **FR-028**: FR-026 and FR-027 MUST hold even if the screen's own checks are bypassed or out of
  date.

### Key Entities

- **Category**: as in feature 002, plus a colour. Defaults have fixed colours; custom categories
  get one when created. Created only in Settings.
- **Account**: as in feature 002. Created only in Settings.
- **Category colour palette**: a set of visible, distinct colours. The first 17 belong to the
  default categories; the rest are given to custom categories.
- **Review row**: as in feature 004. Its category and account fields gain one more invalid state:
  "not one of your categories / accounts".
- **Category slice**: one category's share of the selected month in the donut, drawn in the
  category's colour, with a label (name, amount, share).

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: In any month, 100% of donut slices are visibly coloured and match their legend
  swatches, on both desktop and phone.
- **SC-002**: A category has the same colour in 100% of months; each default category has the
  same colour for 100% of users.
- **SC-003**: The 17 default categories show 17 different colours; a user's first new custom
  categories each get a colour not used by any other of their categories.
- **SC-004**: Hovering, focusing or tapping any slice shows its category name within half a
  second, for 100% of slices.
- **SC-005**: After any number of spendings saved and CSV imports added, the numbers of
  categories and accounts in Settings are unchanged (0 created outside Settings).
- **SC-006**: 100% of unknown category or account fields are highlighted before the user tries to
  save or add, and the save or add is never accepted while one remains.
- **SC-007**: A user can fix an unknown category or account by picking an existing one in under 10
  seconds per field.

## Assumptions

- The hover label replaces reliance on the browser's built-in tooltip, which appears only after a
  delay and not at all on touch screens.
- The colour fix applies to both pie charts: the desktop By category donut (feature 003) and the
  phone top-6 donut (feature 006). The phone donut keeps its top-6 behaviour.
- Users cannot choose or change a category's colour in this fix; colours are assigned. Picking a
  colour in Settings can come later.
- Colours are assigned per user for custom categories; two users' custom categories may share a
  colour.
- Neighbouring slices may share a colour only when a user has more categories than the palette
  has colours; the gap between slices, the legend and the hover label still tell them apart.
- The palette has at least 24 colours, so a user can add several custom categories before any
  colour repeats. The exact colours follow the design.
- The reason texts "Not one of your categories" and "Not one of your accounts" follow the style
  of the existing field messages; the design may adjust the wording.
- Settings already lets the user create, rename and remove custom categories and accounts
  (features 003 and 006); that is unchanged apart from giving new categories a colour.

## Dependencies

- **Feature 002**: categories (with the 17 defaults) and accounts on the server; the rule that a
  missing category or account is created with a spending is removed.
- **Feature 003**: the By category card (Table / Pie), the entry row, autofill, and Settings.
- **Feature 004**: the Review import window, field validity (FR-021, FR-022), the add button
  (FR-031) and all-or-nothing adding (FR-033).
- **Feature 006**: the phone donut, the Add / Edit screen, the review list, the Fix row screen and
  phone Settings.
