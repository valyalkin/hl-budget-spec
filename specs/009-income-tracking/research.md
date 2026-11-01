# Research: Income (feature 009)

Every decision below resolves an open point from [plan.md](./plan.md)'s Technical Context. Code
references are to `hl-budget-backend` and `hl-budget-frontend` at `origin/main` after feature 008.

## R-001: Separate tables for income, not a `kind` column

**Decision**: Flyway V7 adds three tables, `income_categories`, `incomes` and `income_imports`,
shaped like `categories`, `spendings` and `spending_imports`. `incomes` references
`income_categories (id, user_id)` and the existing `accounts (id, user_id)` through composite keys,
exactly as `spendings` does.

**Rationale**:
- The foreign key alone makes FR-013 hold: an income can only point at an income category and a
  spending only at a spending category, with no check in code that could be bypassed.
- Nothing about spendings changes (FR-004): `categories`, `spendings`, their indexes, queries and
  the uniqueness rule (`ux_categories_user_lower_name`) stay as they are. Income category names are
  unique within `income_categories` only (FR-022), so "Other" can exist in both lists.
- Accounts stay one table (FR-050), so both kinds of entry point at the same rows.

**Alternatives considered**:
- *A `kind` column on `categories` and `spendings`*: the spendings table would hold income, every
  existing query and endpoint would need a `kind = 'SPENDING'` filter to stay correct, the unique
  name index would change, and keeping a spending off an income category needs a composite key that
  includes `kind`. More risk to Spendings for less code.
- *A `kind` column on `categories` only*: same index and filter changes, and still no way for the
  foreign key to keep kinds apart.

## R-002: Default income categories and their colours

**Decision**:
- `DefaultIncomeCategories.ALL` = Salary `#1e5b45`, Investments `#1d4ed8` (palette positions 1 and
  3: the brand green and a blue, clearly different from each other, both ≥ 3:1 on white as checked
  in 007).
- They are seeded once per user on the first budgeting call, like the 17 spending defaults, but
  with their own marker `users.income_categories_seeded_at` and their own claim
  (`CurrentUserRepository.claimIncomeSeeding`). Existing users already have `categories_seeded_at`
  set, so reusing that marker would never seed them.
- Custom income categories get `CategoryPalette.next(colour counts over the caller's income
  categories)`. With Salary and Investments taken, the first custom one gets `#e11d48`, then the
  other spare colours.

**Rationale**: the same seeding mechanism and the same colour rule as 002 and 007, so concurrency
behaviour (one seeding even under parallel first calls) is already proven. Colours are counted
within income categories only (spec Edge Cases), as accounts are counted within accounts (008).

**Alternatives considered**: seeding in the V7 migration for existing users. Rejected: new users
still need the runtime path, and two paths for one rule can disagree.

## R-003: API resources

**Decision**: new versioned resources, all under the existing JWT filter and scoped to the caller:

| Resource | Twin of | Notes |
|---|---|---|
| `GET/POST /api/v1/incomes`, `GET/PUT/DELETE /api/v1/incomes/{incomeId}` | `/spendings` | Same paging, filters (`from`, `to`, `categoryId`, `accountId`), sort and `Idempotency-Key`. Field `source` instead of `merchant` |
| `GET/POST /api/v1/income-categories`, `GET/PUT/DELETE /api/v1/income-categories/{id}` | `/categories` | `incomeCount` instead of `spendingCount`; defaults refused on rename and delete with `reason: default-category` |
| `GET /api/v1/income-sources?limit=` | `/merchants` | `incomeCount` instead of `spendingCount` |
| `GET /api/v1/monthly-income-totals?year=` | `/monthly-totals` | `entryCount` per month |
| `GET /api/v1/income-category-totals?from=&to=` | `/category-totals` | Same range rules |
| `GET /api/v1/income-account-totals?from=&to=` | `/account-totals` | Same range rules |
| `POST /api/v1/income-imports`, `GET /api/v1/income-imports/{importId}` | `/spending-imports` | Body `{ incomes: [...] }`, at most 1,000 rows, all or nothing |

Changes to existing resources (all additive):
- `Account` gains `incomeCount` (required in responses, read-only).
- Deleting an account used by income returns the existing `409` with `reason: in-use`;
  `details` keeps `spendingCount` (now possibly 0) and gains `incomeCount`.

**Rationale**: one resource per kind keeps every existing schema and status untouched
(Principle VII), and each new endpoint mirrors one the frontend already calls. New totals schemas
use the neutral `entryCount` rather than reusing `spendingCount` with a different meaning.

**Alternatives considered**: a `?kind=income` parameter on the existing totals and merchants
endpoints. Rejected: their response fields are named `spendingCount`, so the meaning would drift,
and an optional parameter that changes what a resource returns is harder to document and test.

## R-004: Backend code: share the rules, duplicate the thin layers

**Decision**:
- **Shared, generalised**: `SpendingRules` date and amount checks (renamed `EntryRules`, used by
  both services); `TotalsQueries` and `MerchantQueries` take a `Ledger` (a Kotlin enum holding the
  constant table and column names: `spendings`/`spending_date`/`merchant`/`categories` and
  `incomes`/`income_date`/`source`/`income_categories`), so the SQL exists once;
  `NamedItemResolver` takes the category repository to look names up in; `NamedItemErrors.inUse`
  takes a count map.
- **Separate, thin**: JPA entities, repositories, DTOs, controllers and services for income
  (`IncomeEntity`, `IncomeCategoryEntity`, `IncomeImportEntity`, `IncomeService`,
  `IncomeCategoryService`, `IncomeImportService`, `IncomeController`, `IncomeCategoryController`,
  `IncomeImportController`), plus three endpoints added to `TotalsController` and one new
  `IncomeSourceController`.

**Rationale**: the SQL and validation rules are where bugs hide, so they exist once. Entities, DTOs
and controllers carry OpenAPI annotations with different names and examples, so generalising them
would make the published contract harder to read. Identifiers interpolated into SQL come only from
the enum, never from input.

**Alternatives considered**: a generic `Entry<Kind>` hierarchy for entities and controllers.
Rejected: springdoc schema names, JPA mappings and examples all differ per kind, and the
indirection costs more than the duplication at this size (Principle VI).

## R-005: Frontend routing: a `/income` page beside `/spendings`

**Decision**: a new route `app/income/page.tsx` (with `loading.tsx` and `error.tsx`), same
`?month=YYYY-MM` parameter. The desktop header's existing `Main` nav gets a second link,
"Income", and each link carries the current `?month=` (FR-002). On the phone, the header gets a
two-option segmented control "Spendings | Income" made of links. Both use Next's `<Link>`, so
switching is a client-side navigation with no full page reload; `aria-current="page"` marks the
selected tab (FR-095). `/` keeps redirecting to `/spendings` (FR-001).

**Rationale**: separate routes give each tab its own URL, data loader, loading state and error
boundary, and leave `/spendings` exactly as it is. The phone back gesture keeps closing the top sheet first (feature 006's layer stack in
`phone-ui-provider.tsx` pushes one history entry per open layer), so FR-072 holds. Tab links
navigate with `replace`, so Back never lands on the other tab.

**Alternatives considered**: `?tab=income` on `/spendings`. Rejected: one page would load both
kinds' data or branch everywhere, and the Spendings page's tests and loader would change.

## R-006: Frontend components: a `ledger` setting, not copies

**Decision**: a `Ledger = 'spendings' | 'income'` type and a `LEDGER` table
(`app/lib/ledger.ts`) holding everything that differs: headings and nouns ("spendings" / "entries",
"1 entry"), the party column label ("Merchant" / "Source"), empty-month messages, the add button
label ("Add spending" / "Add income"), validation wording ("Add a merchant name." / "Add a
source."), the not-yours category message, the import window title and sample file, and the
breakdown storage keys (`hlb.byCategoryView` and `hlb.byAccountView` unchanged for spendings;
`hlb.income.byCategoryView` and `hlb.income.byAccountView` for income, FR-064).

Components that take a `ledger` prop: `MonthSection`, `MonthLine`, `SpendingsTable`, `EntryRow`,
`SpendingRow`, `MonthTotalCard`, `BreakdownCard`, the import provider and dialogs, and the phone
section, list, form, delete sheet, breakdown and Fix row screen. The UI entry type keeps the
field name `merchant`, documented as "the merchant of a spending, or the source of an income"; the
data layer maps `source ↔ merchant` in one place for income. Default prop value is `'spendings'`,
so existing call sites and tests are unchanged.

**Rationale**: income must behave exactly like Spendings (spec Overview), so the same components
guarantee it, and fixes land in both. Keeping `merchant` as the UI field name avoids renaming
across every Spendings component and test (FR-004), at the cost of one mapping.

**Alternatives considered**: copying the components into `app/ui/income/`. Rejected: two copies
of the entry row's keyboard and autofill logic would drift. Renaming `merchant` to `party`
everywhere: clearer, but a large churn in Spendings code for no user-visible gain.

## R-007: Frontend data and actions

**Decision**:
- `data.ts`: `loadSpendingsPage` becomes `loadLedgerPage(ledger, month)` (with
  `loadSpendingsPage` kept as a thin wrapper), choosing endpoints from a server-side map. For
  income it reads `monthly-income-totals`, `incomes`, `income-category-totals`,
  `income-account-totals`, `income-categories`, `accounts` and `income-sources`, in the same two
  parallel rounds. `getEarliestYear(ledger)` serves the phone Year dropdown.
- `actions.ts`: `saveSpending` / `deleteSpending` / `importSpendings` gain income twins
  (`saveIncome`, `deleteIncome`, `importIncome`) sharing the zod schema and the error mapping;
  `createIncomeCategory` / `deleteIncomeCategory` join the Settings resources. Paths are fixed in
  server code; no client value chooses an endpoint.
- Regenerated `api-schema.d.ts`; new aliases in `definitions.ts` (`Income`, `IncomePage`,
  `IncomeCategory`, `IncomeCategoryList`, `IncomeSourceList`, `IncomeMonthlyTotals`,
  `IncomeCategoryTotals`, `IncomeAccountTotals`, `IncomeImport`).

**Rationale**: the frontend stays a renderer (Principle II): all sums, matching and seeding are on
the server. Separate named actions keep each server action's input narrow and easy to test.

## R-008: Settings and the account rules on screen

**Decision**:
- Desktop and phone Settings get three sections: **Spending categories** (the existing Categories
  panel, renamed), **Income categories** (a new `NamedListPanel` instance with defaults marked and
  colour swatches), **Accounts**.
- Account usage line: unchanged when `incomeCount` is 0 ("3 spendings recorded" / "No spendings
  yet"); otherwise "3 spendings · 1 income entry". Removal refusal: unchanged wording when only
  spendings use the account; otherwise "[Name] has N spendings and M income entries, so it can't be
  removed." (or "… has M income entries …" when N is 0).
- Income category removal refusal: "[Name] is used by N entries, so it can’t be removed." ("1 entry" when N is 1).

**Rationale**: existing wording, and the Spendings e2e that assert it, stay valid for users with
no income (FR-004); the new wording appears only when income is involved.

## R-009: CSV import for income

**Decision**: the same reader (`csv-import.ts`), review table and Fix row screen, with `ledger`
deciding the category list checked against, the labels ("Source"), the sample file
(`public/sample-income.csv`, beside `public/sample-spendings.csv`: Salary and Investments rows) and the action called. Header
matching (`mapColumns`) adds `source|payer|employer` to the merchant pattern, so an income file
with a Source column and a spendings-shaped file both import. The import dialog title reads "Import income" on the Income tab.

**Rationale**: FR-080 to FR-082 with no second parser.

## R-010: Tests and performance

**Decision**:
- Backend: V7 migration IT (tables, constraints, existing data untouched); seeding IT for income
  defaults (new user, pre-existing user, concurrent first calls); `IncomeControllerIT`,
  `IncomeCategoryControllerIT`, `IncomeImportIT`, totals ITs for the three income endpoints,
  `OwnershipIT` cases for every income endpoint (SC-006), `AccountControllerIT` for `incomeCount`
  and the in-use refusal with income, and a performance IT with 10,000 spendings + 1,000 incomes
  (SC-004). OpenAPI export IT covers the new operations.
- Frontend: unit tests for `LEDGER`, the data mapping, actions and the components rendered with
  `ledger="income"`; Playwright: `e2e/income/*.spec.ts` (tabs, month view, add, keyboard and
  autofill, edit/delete, breakdowns, import, Settings income categories, account removal) and
  `e2e/phone/income/*.spec.ts`; axe scans on both new pages. All existing specs stay green
  unchanged.
- Indexes on `incomes` mirror `spendings` (`user_id, income_date DESC, created_at DESC, id`,
  `user_id, category_id`, `user_id, account_id`), which keeps the month queries at the same speed.
