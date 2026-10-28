# Feature Specification: Spending Tracking (Backend)

**Feature Branch**: `002-spending-tracking`

**Created**: 2026-10-27

**Status**: Implemented 2026-10-27 (backend valyalkin/hl-budget-backend#3, frontend types valyalkin/hl-budget-frontend#3; deployed via valyalkin/hl-helm#13, #14, #15). Verified 2026-10-27: Argo CD Synced/Healthy, backend 6-0b3f5c1 Ready on schema version 3, new endpoints answer 401 without a token. Not yet done: a signed-in run of quickstart §3 rows 1, 3 and 5 against the live host

**Input**: User description: "I would like to start building backend for business logic of the budget application. This application is going to be used to record and view spending transactions (spendings). Each spending has date, merchand, category, account and amount. For each user, there are default categories like Rent, Groceries, Public Transport, Car, Household, Health, Beauty, Clothes, Education, Travel, Bills, Subscriptions, Other, Sport, Entertainment, Pet, Taxi, but the user is free to add his own categories. Accounts can also be added/edited/deleted by the user and they should have a name of the account only. All necessary tables should be created. All crud endpoints should be created as well for spendings, accounts and categories. The mentioned above default categories should be available to the user in the beginning. This work is for backend only, frontend will be built in another spec."

## Clarifications

### Session 2026-10-27

- Q: How does currency work? → A: One fixed currency for the whole app, set in the server's
  configuration at deploy time (for example CAD). Every amount is in that currency. Spendings and
  accounts carry no currency of their own. Multiple currencies are out of scope.
- Q: What happens when a user deletes an account or category that spendings still use? → A: The
  delete is refused as a conflict, and the response says how many spendings use it. The user must
  first move those spendings to another account or category, or delete them.
- Q: May users rename or delete the default categories? → A: No. The 17 defaults are fixed:
  they cannot be renamed or deleted. Only categories the user created can be changed.
- Q: Can a spending record a refund or credit (a negative amount)? → A: No. Amounts are always
  positive (greater than zero). Refunds and credits are out of scope for this feature.
- Q: Are spendings dated in the future accepted? → A: No. The date must be today or earlier,
  where "today" is taken in the app's configured time zone.
- Q: Should the account and category lists show how many spendings use each item? → A: Yes.
  Each account and category, in lists and when fetched alone, includes the number of spendings
  that use it (read-only, never set by the user).

## Scope

This feature delivers the server side of HL Budget's first business capability: recording and
viewing spendings, and managing the accounts and categories they belong to. It adds the stored
data and the versioned server interface (create, read, update, delete) for **spendings**,
**accounts** and **categories**, all scoped to the signed-in user.

The screens that use this interface are a separate, later feature. In this feature, "the user"
means a signed-in caller of the server interface (in practice, the future web app or a test
client), identified by the sign-in built in feature 001.

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Start with ready-made categories (Priority: P1)

A signed-in user who has never used the budgeting features asks for their categories and gets
the 17 default categories: Rent, Groceries, Public Transport, Car, Household, Health, Beauty,
Clothes, Education, Travel, Bills, Subscriptions, Other, Sport, Entertainment, Pet, Taxi. They do
not have to set anything up first. Each user gets these defaults, and what one user does to their
categories never affects another user.

**Why this priority**: Every spending needs a category. Ready-made categories let a user record
their first spending at once, and the user explicitly asked for them to be there from the start.

**Independent Test**: As a brand-new user, list categories and confirm exactly the 17 defaults
are returned. As a second new user, confirm they get their own 17 defaults. As a user who already
existed before this feature, confirm they get the defaults too.

**Acceptance Scenarios**:

1. **Given** a signed-in user who has never used the budgeting features, **When** they list
   their categories, **Then** they receive exactly the 17 default categories, each marked as a
   default and each used by 0 spendings.
2. **Given** a user who signed in before this feature was released, **When** they list their
   categories for the first time, **Then** they also receive the 17 defaults.
3. **Given** two different users, **When** each lists categories, **Then** each sees only their
   own categories, and a change by one is never visible to the other.
4. **Given** a user who has already received their defaults, **When** they list categories
   again, **Then** the defaults are not added a second time.

---

### User Story 2 - Manage my accounts (Priority: P1)

A user creates the accounts they pay from, such as "Amex", "Debit card" or "Cash". An account has
only a name. The user can list their accounts, view one, rename it and delete it.

**Why this priority**: Every spending needs an account, and there are no default accounts, so a
user cannot record a spending until they have created at least one.

**Independent Test**: Create two accounts, list them, rename one, delete the other, and confirm
the list reflects each change. Confirm another user cannot see or change them.

**Acceptance Scenarios**:

1. **Given** a signed-in user, **When** they create an account named "Amex", **Then** the
   account is saved and returned with its identifier and name.
2. **Given** a user with accounts, **When** they list their accounts, **Then** they see all of
   their accounts and only theirs, sorted by name, each with the number of spendings that use it
   (for example "Amex" with 2, "Cash" with 0).
3. **Given** a user with an account "Amex", **When** they rename it to "Amex Gold", **Then** the
   new name is saved, and spendings on that account show the new name.
4. **Given** a user with an account "Amex", **When** they try to create a second account named
   "amex", **Then** the request is rejected because the name is already in use.
5. **Given** a user with an account no spending uses, **When** they delete it, **Then** it no
   longer appears in their list.
6. **Given** an account that spendings still use, **When** the user tries to delete it, **Then**
   the delete is refused, the response says how many spendings use it, and nothing changes.
7. **Given** an account that belongs to another user, **When** a user tries to view, rename or
   delete it, **Then** the server responds as if the account does not exist.

---

### User Story 3 - Record a spending (Priority: P1)

A user records a spending by giving its date, merchant, amount, category and account. The server
checks the details, saves the spending and returns it. The user can later view it, correct any of
its details, or delete it.

**Why this priority**: Recording spendings is the purpose of the application. Stories 1 and 2
exist to make this possible.

**Independent Test**: With one account and the default categories, create a spending, fetch it by
its identifier, change its amount and category, and delete it, checking the stored values after
each step.

**Acceptance Scenarios**:

1. **Given** a user with an account and categories, **When** they create a spending dated
   2026-10-20 at merchant "Loblaws" for 84.37 in "Groceries" on "Amex", **Then** it is saved and
   returned with its identifier, all its details, and when it was created.
2. **Given** a saved spending, **When** the user fetches it by identifier, **Then** they receive
   the same details, including the names of its category and account.
3. **Given** a saved spending, **When** the user changes its amount, date, merchant, category or
   account, **Then** only the changed details differ and the change is saved.
4. **Given** a saved spending, **When** the user deletes it, **Then** it can no longer be fetched
   or listed.
5. **Given** a create or update request with a missing or invalid detail (no date, empty
   merchant, date in the future, zero or negative amount, more than 2 decimal places, unknown
   category or account),
   **Then** nothing is saved and the response names every invalid field.
6. **Given** a category or account that belongs to another user, **When** a user tries to use it
   on a spending, **Then** the request is rejected as if that category or account does not
   exist.
7. **Given** a create request that is sent twice with the same idempotency key (for example
   because of a network retry), **When** both requests reach the server, **Then** only one
   spending is created and both responses return it.

---

### User Story 4 - View my spendings (Priority: P1)

A user lists their spendings, newest first, one page at a time. They can narrow the list to a
date range, one or more categories, or one or more accounts, and they can sort by date or amount.

**Why this priority**: "View" is half of the stated purpose. Without filtering by date range, the
list becomes unusable after a few months of data.

**Independent Test**: Create 30 spendings across two months, two accounts and three categories.
Check that the first page holds the newest spendings, that paging returns every spending exactly
once, and that each filter returns exactly the matching spendings.

**Acceptance Scenarios**:

1. **Given** a user with spendings, **When** they list spendings without filters, **Then** they
   receive their spendings newest date first, in pages, with the total number of matching
   spendings and pages.
2. **Given** a user with spendings in September and October, **When** they list with a date range
   of 2026-10-01 to 2026-10-31, **Then** only October spendings are returned, and both end dates
   are included.
3. **Given** a user with spendings in several categories, **When** they filter by "Groceries" and
   "Taxi", **Then** only spendings in those two categories are returned. Filtering by account
   works the same way.
4. **Given** a user who asks for sorting by amount, largest first, **When** the list is returned,
   **Then** it is in that order, and spendings with the same amount keep a stable order between
   pages.
5. **Given** a user who has no spendings, or whose filters match nothing, **When** they list
   spendings, **Then** they receive an empty page, not an error.
6. **Given** two users, **When** either lists spendings, **Then** they never see the other's
   spendings.

---

### User Story 5 - Tailor my categories (Priority: P2)

A user adds their own categories (for example "Coffee" or "Gifts"), renames categories, and
deletes categories they do not use.

**Why this priority**: The defaults cover common spending, so a user can work without custom
categories at first, but the user explicitly asked to be able to add their own.

**Independent Test**: Add a category "Coffee", record a spending in it, rename it to "Cafés", and
confirm the spending shows the new name. Delete an unused category and confirm it is gone from
the list.

**Acceptance Scenarios**:

1. **Given** a signed-in user, **When** they create a category named "Coffee", **Then** it is
   saved, appears in their category list, and is marked as not a default.
2. **Given** a user with a category "Groceries", **When** they try to create another category
   named "groceries", **Then** the request is rejected because the name is already in use.
3. **Given** a user with a category, **When** they rename it, **Then** the new name is saved,
   and spendings in that category show the new name.
4. **Given** a user with a category they created that no spending uses, **When** they delete
   it, **Then** it no longer appears in their list.
5. **Given** a default category such as "Groceries", **When** the user tries to rename or delete
   it, **Then** the request is rejected and the category is unchanged.
6. **Given** a category the user created that spendings still use, **When** they try to delete
   it, **Then** the delete is refused, the response says how many spendings use it, and nothing
   changes.

---

### Edge Cases

- **Deleting an account or category that spendings still use**: refused as a conflict, with the
  number of spendings that use it. If the last spending using it is moved or deleted, the delete
  then succeeds. A spending can never end up without an account or category.
- **Custom category with a default's name**: creating "groceries" when the default "Groceries"
  exists is rejected as a duplicate name, like any other duplicate.
- **Names**: account and category names are trimmed of surrounding spaces, must not be empty
  after trimming, may be at most 50 characters, and must be unique per user regardless of letter
  case. Renaming an item to its own current name (or only changing its letter case) is allowed.
- **Merchant**: required, trimmed, at most 255 characters. It is free text; the same merchant can
  be spelled differently on different spendings.
- **Amount**: must be greater than zero, have at most 2 decimal places, and be at most
  99,999,999.99. It is stored and returned exactly, with no rounding drift.
- **Date**: a valid calendar date, today or earlier. Past dates of any age are allowed (for
  back-filling from statements). A date after today is rejected as invalid. "Today" is taken in
  the app's configured time zone, so a spending entered late in the evening is not refused
  because the server clock is already on the next day in UTC.
- **Invalid filters**: a date range whose start is after its end, an unknown sort field, or a page
  size above the maximum is rejected with a message naming the problem.
- **Unknown identifiers**: fetching, changing or deleting a spending, account or category that
  does not exist, or belongs to another user, gives the same "not found" result, so nobody can
  tell whether another user's item exists.
- **Concurrent edits**: if two updates to the same item arrive together, neither is half-applied.
  The second one either succeeds on top of the first or is rejected as a conflict.
- **Deleting twice**: deleting an item that was already deleted gives "not found" and changes
  nothing.
- **Defaults and existing users**: users created before this feature receive the defaults the
  first time they use any budgeting feature, exactly once, even if two requests arrive at the
  same moment.

## Requirements *(mandatory)*

### Functional Requirements

**General**

- **FR-001**: Every spending, account and category MUST belong to exactly one user, the one who
  created it (or, for default categories, the one they were created for).
- **FR-002**: A user MUST only be able to see, use, change or delete their own spendings,
  accounts and categories. An item belonging to someone else MUST be reported as not found.
- **FR-003**: Only signed-in users MUST be able to use these features; the user's identity MUST
  come from the verified sign-in, never from data in the request.
- **FR-004**: The server MUST offer create, list, fetch-one, update and delete operations for
  spendings, accounts and categories, as part of the versioned server interface defined in
  feature 001.
- **FR-005**: Every invalid request MUST be rejected without saving anything, and the response
  MUST name each invalid field and why it is invalid.
- **FR-006**: All data MUST be stored durably and survive a restart of the server.

**Categories**

- **FR-010**: Each user MUST have the 17 default categories (Rent, Groceries, Public Transport,
  Car, Household, Health, Beauty, Clothes, Education, Travel, Bills, Subscriptions, Other, Sport,
  Entertainment, Pet, Taxi) available from their first use of the budgeting features, without any
  setup step.
- **FR-011**: Defaults MUST be given to each user exactly once, including users who existed
  before this feature, and MUST never be duplicated.
- **FR-012**: Users MUST be able to create categories with a name of their choosing.
- **FR-013**: Category names MUST be unique per user, ignoring letter case and surrounding
  spaces, non-empty, and at most 50 characters.
- **FR-014**: Each category MUST show whether it is a default category.
- **FR-015**: Users MUST be able to rename and delete the categories they created. Default
  categories MUST NOT be renamed or deleted; such requests MUST be rejected without changing
  anything.
- **FR-016**: The category list MUST return all of the user's categories in one response, sorted
  alphabetically by name.

**Accounts**

- **FR-020**: Users MUST be able to create, list, fetch, rename and delete accounts.
- **FR-021**: An account MUST have a name and no other user-editable details.
- **FR-022**: Account names MUST follow the same rules as category names (unique per user
  ignoring case, non-empty, at most 50 characters).
- **FR-023**: No accounts are created automatically; a new user starts with none.
- **FR-024**: The account list MUST return all of the user's accounts in one response, sorted
  alphabetically by name.
- **FR-025**: Deleting an account or category that any spending still uses MUST be refused as a
  conflict, MUST report how many spendings use it, and MUST change nothing.
- **FR-026**: Every account and category returned (in a list or fetched alone) MUST include the
  number of the user's spendings that use it. The count is always current, is read-only, and is
  never accepted from the request.

**Spendings**

- **FR-030**: A spending MUST have a date, a merchant, an amount, one category and one account,
  all required. The date MUST NOT be later than today in the app's configured time zone; a
  future date MUST be rejected as an invalid field.
- **FR-031**: The category and account of a spending MUST belong to the same user as the
  spending.
- **FR-032**: The amount MUST be a positive value with at most 2 decimal places, stored and
  returned exactly. All amounts are in the single app currency set in the server's configuration,
  and every returned amount MUST state that currency.
- **FR-033**: Users MUST be able to change any detail of a saved spending, and to delete it.
- **FR-034**: When a spending is returned, it MUST include the identifier and current name of
  its category and account, so the reader does not need extra lookups to show them.
- **FR-035**: Each spending MUST record when it was created and when it was last changed.
- **FR-036**: Creating a spending MUST accept an optional idempotency key. Repeating a create
  with the same key MUST return the spending created the first time and MUST NOT create another.
- **FR-037**: The spending list MUST be paged, newest date first by default, with a page size of
  20 by default and at most 100, and MUST report the total number of matching spendings and pages.
- **FR-038**: The spending list MUST support filtering by an inclusive date range (start, end or
  both), by one or more categories, and by one or more accounts. Filters combine with AND.
- **FR-039**: The spending list MUST support sorting by date or amount, ascending or descending,
  with a stable tie-break so paging never skips or repeats a spending.

**Interface quality**

- **FR-040**: Every new operation, its inputs, its outputs and its error responses MUST be
  described in the published interface document with examples, so the frontend feature can be
  built from that document alone.
- **FR-041**: The changes MUST be additions to the existing interface version; nothing that
  feature 001 published may change or break.

### Key Entities *(include if feature involves data)*

- **User** (exists from feature 001): the signed-in person. Owns all accounts, categories and
  spendings below.
- **Account**: where money was paid from (a card, a bank account, cash). Has a name and belongs
  to one user. Referenced by many spendings. Shown with its spending count (derived, not stored
  or edited by the user).
- **Category**: what the money was spent on. Has a name, a flag saying whether it is one of the
  defaults, and belongs to one user. Referenced by many spendings. Shown with its spending count
  (derived, not stored or edited by the user).
- **Spending**: one spending transaction. Has a date, a merchant, an amount, one category, one
  account, a created time and a last-changed time, and belongs to one user.
- **Default category list**: the fixed list of 17 category names given to every user at the
  start. It is part of the product. Users cannot rename or delete these categories.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: A brand-new user can record their first spending in 2 requests (create an account,
  then create the spending), with no category setup.
- **SC-002**: 100% of new and pre-existing users see exactly 17 default categories on first use,
  with no duplicates, including when several first requests arrive at once.
- **SC-003**: In automated tests, no request by one user can read, change, delete or reference
  another user's spending, account or category (0 cross-user leaks across every operation).
- **SC-004**: For a user with 10,000 spendings, a filtered page of the spending list returns in
  under 1 second.
- **SC-005**: Amounts read back are identical to amounts saved in 100% of cases, and totals of
  saved amounts show no rounding error.
- **SC-006**: Every operation in this feature is documented in the published interface document,
  so the frontend feature needs no other source to call it.
- **SC-007**: Repeating a spending create with the same idempotency key 10 times creates exactly
  one spending.

## Assumptions

- **Backend only**: no screens are built here. The web app feature that follows will use this
  interface. The existing home page is unchanged.
- **Spendings only**: income, transfers between accounts, budgets, recurring spendings, and
  importing statements are out of scope. Refunds are out of scope too: amounts are always
  positive.
- **No reports**: totals, summaries and charts by category or month are out of scope and will
  get their own feature. The list does report how many spendings match.
- **One user per data set**: there is no sharing of accounts or categories between users and no
  household or family mode.
- **No search**: free-text search by merchant is out of scope; filtering is by date, category and
  account only.
- **One currency**: all amounts are in one currency set at deploy time. Changing it later does
  not convert existing amounts. Multi-currency support is out of scope.
- **One time zone**: the app has one time zone, set in the server's configuration alongside the
  currency, used only to decide what "today" is for the future-date check.
- **Hard delete**: deleting a spending removes it permanently; there is no undo or trash.
- **Category lists stay small**: users have tens of categories and accounts, not thousands, so
  those lists are returned whole, without paging.
- **Defaults are created on first use**: the defaults are added the first time a user uses any
  budgeting feature, which covers users created before this feature as well as new ones.
- **Default names are in English**, exactly as listed above. Translation is out of scope.
- **Sign-in and user records** come from feature 001 and are reused unchanged.
