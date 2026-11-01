---

description: "Task list for 009-income-tracking"
---

# Tasks: Income

**Input**: Design documents from `/specs/009-income-tracking/`

**Prerequisites**: plan.md, spec.md, research.md, data-model.md, contracts/openapi-delta.yaml,
quickstart.md

**Tests**: REQUIRED. Constitution Principle IV: tests ship with the code, and every new
user-visible flow gets a Playwright spec. Within each phase, write the tests first and see them
fail before writing the code that satisfies them.

**Organization**: Tasks are grouped by user story (spec.md US1–US6), in priority order: US1, US2
(P1), US3, US4, US6 (P2), US5 (P3). All backend work is foundational, because every story reads
it, and the backend can ship on its own (additive API). The frontend foundation threads a `ledger`
setting through the shared components with `'spendings'` as the default, so each story only wires
up its own income screen.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Can run in parallel (different files, no dependency on an unfinished task).
- **[Story]**: The user story the task belongs to (US1 … US6).
- Every task ends with `· skill: <name>`. Load that skill before starting; its definition of done
  is part of the task's done. Where a skill and the constitution disagree, the constitution wins.

## Path Conventions

All paths are relative to the spec repo root `/Users/valyalkin/Documents/Projects/hl-budget-spec/`.

- `hl-budget-backend/`: git worktree of `valyalkin/hl-budget-backend`, on branch
  `009-income-tracking` (T001).
  - Main code: `hl-budget-backend/src/main/kotlin/com/valyalkin/hlbudget/`, written `…/main/`.
  - Tests: `hl-budget-backend/src/test/kotlin/com/valyalkin/hlbudget/`, written `…/test/`.
  - Migrations: `hl-budget-backend/src/main/resources/db/migration/`, written `…/migration/`.
- `hl-budget-frontend/`: git worktree of `valyalkin/hl-budget-frontend`, on branch
  `009-income-tracking` (T001). Written `fe/`.
- **Commits**: inside the service repos, Conventional Commits (`feat:`, `test:`, `refactor:`),
  after each task or logical group.
- **Merging**: merging to `main` in a service repo triggers a release. Do it only after the gate
  is green and the user agrees.

Shared rules that several tasks reference:

- **Regenerate types**: run `./gradlew check` in `hl-budget-backend/`; `OpenApiExportIT` rewrites
  `hl-budget-backend/contracts/api.yaml`. Then
  `cp hl-budget-backend/contracts/api.yaml hl-budget-frontend/contracts/api.yaml` and, in `fe/`,
  `pnpm generate:api`. Commit both frontend files.
- **Twin rule** (research R-003, R-004): an income piece mirrors its spending twin's behaviour,
  status codes, messages, pointers and OpenAPI style exactly, except where this file or the
  contract says otherwise. Twins: `incomes` ↔ `spendings`, `income_categories` ↔ `categories`,
  `income_imports` ↔ `spending_imports`, `source` ↔ `merchant`, `income-sources` ↔ `merchants`,
  `incomeCount` ↔ `spendingCount` (on categories, sources, imports), `entryCount` ↔
  `spendingCount` (on totals).
- **Default income categories** (research R-002): Salary `#1e5b45`, Investments `#1d4ed8`, the
  same for every user, `is_default = true`, never renamed or deleted. A custom income category gets
  `CategoryPalette.next(counts)` where `counts` are the caller's **income categories** per colour
  (spending categories and accounts are not counted), so the first custom one is `#e11d48`, then
  `#0891b2`, `#a855f7`, `#365314`, `#1e3a8a`, `#701a75`, `#ea580c`.
- **Not yours = not found** (backend): every new query filters on the caller's `userId`; another
  user's income or income category is `404`, and naming one in a request is `422 NotFound`.
- **Spendings must not change** (FR-004): `/spendings`, its components' default behaviour, the
  localStorage keys `hlb.byCategoryView` and `hlb.byAccountView`, every existing test id and text,
  and every existing unit and e2e test pass unchanged (only import paths may change). New props
  default to the spending behaviour.
- **Client components**: no `useMemo`, `useCallback` or `React.memo` (the React Compiler is on).

---

## Phase 1: Setup

**Purpose**: Fresh feature branches with green gates.

- [X] T001 In both `hl-budget-backend/` and `hl-budget-frontend/`:
  1. `git fetch origin` and `git switch -c 009-income-tracking origin/main` (both worktrees are on
     `008-account-breakdown`, already merged; if there is uncommitted work, ask the user).
  2. Confirm the gates are green before any change: `./gradlew check` in the backend;
     `pnpm install && pnpm typecheck && pnpm lint && pnpm test` in the frontend.

  The spec repo is already on `009-income-tracking`; ask the user before committing
  `specs/009-income-tracking/` · skill: none

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: The backend stores and serves income, income categories, income totals, sources and
imports, and counts income on accounts; the frontend has the types, the `LEDGER` table, income
data readers and actions, and ledger-aware components. No visible change yet.

### Backend tests first (see them fail)

- [X] T002 [P] Create `…/test/repository/IncomeMigrationIT.kt`, modelled on
  `AccountColorMigrationIT.kt`: migrate to V6 (`target("6")`), insert a user with a category, an
  account and a spending; migrate to latest and assert: the spending, category and account are
  unchanged; `users.income_categories_seeded_at` exists and is NULL; `income_categories`,
  `incomes`, `income_imports` exist; inserting an income whose `category_id` is a **spending**
  category fails the FK; an income with `amount = 0`, a source `' x'` (not trimmed), or a key
  without fingerprint fails its check; two income categories "Salary" and "salary" for one user
  fail `ux_income_categories_user_lower_name`, while an income category named like an existing
  spending category succeeds; `color = 'red'` fails `ck_income_categories_color`; deleting an
  account used by an income fails · skill: backend-developer
- [X] T003 [P] Create `…/test/controller/DefaultIncomeCategoriesIT.kt`, modelled on
  `DefaultCategoriesIT.kt`: a new user's first `GET /api/v1/income-categories` → exactly
  Investments `#1d4ed8` and Salary `#1e5b45` (sorted by name), `isDefault: true`, `incomeCount: 0`;
  a user whose `categories_seeded_at` is already set (pre-009) also gets them, once; 10 concurrent
  first calls → still exactly 2 rows; the 17 spending categories are not affected · skill:
  backend-developer
- [X] T004 [P] Create `…/test/controller/IncomeCategoryControllerIT.kt`, modelled on
  `CategoryControllerIT.kt` (quickstart §3.2–3.3): create "Other" → `201` with `Location`, colour
  `#e11d48`, even though a spending category "Other" exists; then "Freelance" → `#0891b2`; "salary"
  → `409 duplicate-name`; empty or 51-character name → `422`; rename or delete Salary → `422`
  `reason: default-category`; rename "Other" keeps its colour; delete an unused custom one → `204`;
  delete one used by an income → `409 in-use` with `details.incomeCount`; `GET
  /api/v1/categories` never lists income categories and `GET /api/v1/income-categories` never lists
  spending ones · skill: backend-developer
- [X] T005 [P] Create `…/test/service/IncomeCategoryServiceTest.kt`, modelled on
  `CategoryServiceTest.kt`: colour from `next` over income-category colour counts only; defaults
  refused on rename/delete; duplicate pre-check and lost-race path · skill: backend-developer
- [X] T006 [P] Create `…/test/controller/IncomeControllerIT.kt`, covering what
  `SpendingControllerIT.kt`, `SpendingByNameIT.kt`, `SpendingListIT.kt` and `IdempotencyIT.kt`
  cover for spendings (quickstart §3.4–3.5):
  - create by ids and by names (`categoryName: "salary"` matches Salary) → `201`, `Location`,
    body with `source`, `category`, `account`, `amount` in CAD;
  - same `Idempotency-Key` twice → one income, same body; same key, different body → `409`;
  - `categoryName: "Groceries"` (a spending category) → `422` `#/categoryName` "is not one of your
    income categories"; unknown `accountName` → `422` `#/accountName` "is not one of your
    accounts"; both ids and names → `422` `ExactlyOne`; nothing is created in any case;
  - future date, `amount` `"0"` or `"1.234"`, blank or 256-character `source` → `422` with the
    spendings' pointers and messages, all reported at once;
  - list: paging (`size` 1–100), filters `from`, `to`, `categoryId`, `accountId`, sort by date and
    amount with stable ties; GET one, PUT replace, DELETE → `204`, then `404`;
  - `POST /api/v1/spendings` with `categoryName: "Salary"` → `422 NotFound` (FR-003).
  · skill: backend-developer
- [X] T007 [P] Extend `…/test/controller/OwnershipIT.kt` (done as a new `IncomeOwnershipIT.kt`): user B gets `404` for user A's income
  (GET, PUT, DELETE) and income category (GET, PUT, DELETE) and income import receipt; B naming or
  passing the id of A's income category or account in an income or income import → `422 NotFound`;
  none of A's incomes appear in B's income list, sources or any income totals (SC-006) · skill:
  backend-developer
- [X] T008 [P] Extend `…/test/controller/AccountControllerIT.kt` and `DeleteInUseIT.kt` (done as a new `AccountIncomeIT.kt`)
  (quickstart §3.8): `GET /api/v1/accounts` and `/accounts/{id}` include `incomeCount`; deleting an
  account used only by 2 incomes → `409 in-use` with `details` `{ reason: "in-use",
  spendingCount: 0, incomeCount: 2 }`; used by both → both counts; the existing spending-only
  refusal body still has `spendingCount` and now `incomeCount: 0`; after deleting the incomes the
  account can be deleted (unless it is the last one, `LastAccountIT` unchanged) · skill:
  backend-developer
- [X] T009 [P] Create `…/test/controller/IncomeTotalsIT.kt`, modelled on `TotalsIT.kt`
  (quickstart §3.6): October incomes Salary 5000.00 on Debit card, Investments 120.25 and 200.25 on
  Brokerage, plus a September Salary → `GET /api/v1/monthly-income-totals?year=2026` has twelve
  months, October `5320.50` with `entryCount: 3`, months without income `0.00`/0;
  `/income-category-totals?from=2026-10-01&to=2026-10-31` → [Salary 5000.00 count 1 `#1e5b45`,
  Investments 320.50 count 2 `#1d4ed8`]; `/income-account-totals` → [Debit card 5000.00, Brokerage
  320.50] with each account's colour; totals equal the sum of items; spendings in October change
  none of these and incomes change none of `/monthly-totals`, `/category-totals`,
  `/account-totals`; empty month → `items: []`; range errors as `TotalsIT` (pointers `#/from`,
  `#/to`); no token → `401` · skill: backend-developer
- [X] T010 [P] Create `…/test/controller/IncomeSourcesIT.kt`, modelled on `MerchantsIT.kt`
  (quickstart §3.7): "Acme Corp" ×3 and "acme corp " ×1 → one source, `incomeCount: 4`, name and
  last category/account from the most recent entry; merchants never appear in `/income-sources` and
  sources never appear in `/merchants`; `limit` 1–1000, `truncated` · skill: backend-developer
- [X] T011 [P] Create `…/test/controller/IncomeImportsIT.kt`, modelled on `SpendingImportsIT.kt`
  (quickstart §3.9): 2 valid rows → `201`, receipt `incomeCount: 2`, `firstDate`, `lastDate`,
  `total`; a row with `categoryName: "Groceries"` → `422` at `#/incomes/1/categoryName`, nothing
  saved; 1001 rows → `422`; repeated key → same receipt, no new rows; `GET
  /api/v1/income-imports/{id}` → receipt · skill: backend-developer
- [X] T012 [P] Create `…/test/controller/IncomePerformanceIT.kt`, modelled on
  `TotalsPerformanceIT.kt`: seed 10,000 spendings and 1,000 incomes for one user; the month's
  income list, the three income totals and income sources each answer within the existing
  performance limit (SC-004) · skill: backend-developer
- [X] T013 [P] Unit tests (IncomeService and the ledger paths of TotalsService are covered by the ITs; a mocked IncomeServiceTest was not added): rename `…/test/service/SpendingRulesTest.kt` → `EntryRulesTest.kt`
  (same cases); create `…/test/service/IncomeServiceTest.kt` modelled on `SpendingServiceTest.kt`;
  extend `NamedItemResolverTest.kt` with income-category resolution and the "is not one of your
  income categories" message; extend `CurrentUserServiceTest.kt`: income seeding claimed once,
  independent of `categories_seeded_at`; extend `TotalsServiceTest.kt` with the three income
  methods (same `checkRange`) · skill: backend-developer

### Backend implementation

- [X] T014 Create `…/migration/V7__create_incomes.sql` exactly as data-model.md, with a header
  comment like V3's pointing to `specs/009-income-tracking/data-model.md`:
  - `ALTER TABLE users ADD COLUMN income_categories_seeded_at timestamptz NULL;`
  - `income_categories`: "`name varchar(50) NOT NULL` — 1 to 50 characters, already trimmed",
    "`is_default boolean NOT NULL DEFAULT false`", "`color varchar(7) NOT NULL`, `^#[0-9a-f]{6}$`,
    no default", `uq_income_categories_id_user UNIQUE (id, user_id)`, `CREATE UNIQUE INDEX
    ux_income_categories_user_lower_name ON income_categories (user_id, lower(name))`;
  - `incomes`: "`income_date date NOT NULL`", "`source varchar(255) NOT NULL` — 1 to 255
    characters, already trimmed (`ck_incomes_source`)", "`amount numeric(10,2) NOT NULL` — `> 0`",
    FK `(category_id, user_id) → income_categories(id, user_id) ON DELETE NO ACTION`, FK
    `(account_id, user_id) → accounts(id, user_id) ON DELETE NO ACTION`, the idempotency pair
    check, and indexes `ux_incomes_user_idempotency_key`, `ix_incomes_user_date (user_id,
    income_date DESC, created_at DESC, id)`, `ix_incomes_user_category`, `ix_incomes_user_account`;
  - `income_imports`: as `spending_imports` with `income_count integer` "1 to 1000", `first_date
    <= last_date`, "`total numeric(14,2) > 0`", unique `(user_id, idempotency_key)` where set.
  · skill: backend-developer
- [X] T015 Shared refactor (research R-004), keeping every existing spending test green:
  - create `…/main/model/Ledger.kt`: `enum class Ledger(val table: String, val dateColumn:
    String, val partyColumn: String, val categoryTable: String)` with `SPENDINGS("spendings",
    "spending_date", "merchant", "categories")` and `INCOMES("incomes", "income_date", "source",
    "income_categories")`; KDoc: identifiers come only from this enum, never from input;
  - `…/main/repository/spending/TotalsQueries.kt` and `MerchantQueries.kt`: every method takes a
    `ledger: Ledger` and builds its SQL from the enum's names; existing callers pass
    `Ledger.SPENDINGS`;
  - `git mv …/main/service/SpendingRules.kt EntryRules.kt` (class renamed, behaviour unchanged);
  - `…/main/service/NamedItemErrors.kt`: `inUse(resource, id, counts: Map<String, Long>, cause)`
    putting every count into `details` and wording "is used by N spendings and M income entries"
    (only the non-zero parts); existing callers pass `mapOf("spendingCount" to n)` except accounts
    (T019).
  · skill: backend-developer
- [X] T016 Income categories:
  - `…/main/model/DefaultIncomeCategories.kt` (Salary `#1e5b45`, Investments `#1d4ed8`, reusing
    `DefaultCategory`) and `…/main/model/IncomeCategory.kt` (`id`, `name`, `isDefault`,
    `incomeCount`, `color`);
  - `…/main/repository/incomecategory/IncomeCategoryEntity.kt` (`color` `updatable = false`),
    `IncomeCategoryRepository.kt` (row with `incomeCount` subquery, `colorCounts`,
    `findByUserIdAndNameIgnoreCase`, `existsBy…`, `lockByIdAndUserId`, `findAllByUserId`),
    mappings;
  - `…/main/repository/CurrentUserRepository.kt`: `CurrentUserRow.incomeCategoriesSeededAt`,
    `claimIncomeSeeding(userId)` (conditional `UPDATE users SET income_categories_seeded_at = now()
    WHERE id = :id AND income_categories_seeded_at IS NULL`), `insertDefaultIncomeCategories`;
    `…/main/service/CurrentUserService.kt` seeds them after the spending defaults, in the same
    transaction;
  - `…/main/service/IncomeCategoryService.kt` (twin of `CategoryService`, resource name
    `income-category`, in-use count from incomes);
  - `…/main/dto/incomecategory/` (`IncomeCategoryResponse`, `IncomeCategoryListResponse`,
    mappings) and `…/main/controller/IncomeCategoryController.kt` at
    `/api/v1/income-categories`, every operation, parameter, response and example as in
    `contracts/openapi-delta.yaml`, reusing `NameRequest`.
  · skill: backend-developer
- [X] T017 Incomes:
  - `…/main/model/Income.kt`, `IncomeCommand.kt`;
  - `…/main/repository/income/IncomeEntity.kt`, `IncomeRepository.kt` (`countByUserIdAndCategoryId`,
    `countByUserIdAndAccountId`, idempotency lookup), `IncomeSpecifications.kt`, `IncomeInserts.kt`,
    mappings, all twins of the spending ones;
  - `…/main/service/NamedItemResolver.kt`: resolve the category against income categories when
    resolving an income (constructor gains `IncomeCategoryRepository`; a `Ledger` parameter on
    `resolve` and `names`), message "is not one of your income categories";
  - `…/main/service/IncomeService.kt` (twin of `SpendingService`: `EntryRules`, resolver,
    idempotent create with fingerprint, list/get/replace/delete);
  - `…/main/dto/income/` (`IncomeRequest` with "`source` 1 to 255 characters" and the same
    category/account id-or-name fields, `IncomeResponse`, `IncomePageResponse`, mappings) and
    `…/main/controller/IncomeController.kt` at `/api/v1/incomes`, documented as in the contract.
  · skill: backend-developer
- [X] T018 Income totals and sources: `…/main/model/Totals.kt` gains `IncomeMonthTotal`,
  `IncomeMonthlyTotals`, `IncomeCategoryTotal`, `IncomeCategoryTotals`, `IncomeAccountTotal`,
  `IncomeAccountTotals`, `IncomeSource`, `IncomeSourceList`; `…/main/service/TotalsService.kt`
  gains `monthlyIncome`, `incomeByCategory`, `incomeByAccount` using `Ledger.INCOMES`;
  `…/main/service/IncomeSourceService.kt`; `…/main/dto/totals/TotalsResponses.kt` gains the six
  income schemas (with `entryCount`) and `…/main/dto/income/IncomeSourceResponses.kt`;
  `…/main/controller/TotalsController.kt` gains `getMonthlyIncomeTotals`,
  `listIncomeCategoryTotals`, `listIncomeAccountTotals`; new
  `…/main/controller/IncomeSourceController.kt` (`GET /api/v1/income-sources`, `limit` 1–1000,
  default 500). Paths, operation ids and examples as in the contract · skill: backend-developer
- [X] T019 Accounts count income: `…/main/repository/account/AccountRepository.kt` `AccountRow`
  gains `incomeCount` (subquery on `IncomeEntity`) in both counted queries; `…/main/model/Account.kt`,
  mappings and `…/main/dto/account/AccountResponse.kt` gain `incomeCount` (`READ_ONLY`, minimum 0,
  description "How many of the caller's income entries use this account");
  `…/main/service/AccountService.kt` `requireUnused` refuses when spendings **or** incomes use it,
  with `inUse(…, mapOf("spendingCount" to s, "incomeCount" to i))`; `create` returns
  `incomeCount = 0` · skill: backend-developer
- [X] T020 Income imports: `…/main/model/IncomeImport.kt`, `…/main/repository/incomeimport/`
  (entity, repository, inserts), `…/main/service/IncomeImportService.kt` (twin of
  `SpendingImportService`: every row checked, names matched against income categories and
  accounts, all or nothing, idempotent), `…/main/dto/incomeimport/` (request with `incomes` "1 to
  1000" rows, response with `incomeCount`), `…/main/controller/IncomeImportController.kt` at
  `/api/v1/income-imports`, documented as in the contract · skill: backend-developer
- [X] T021 Run `./gradlew check` (T002–T013 now pass; every pre-009 test unchanged and green;
  Kover floors hold, incl. 75% line per new package), then **Regenerate types**. Check the exported
  `contracts/api.yaml` contains every operation id in `contracts/openapi-delta.yaml` and
  `Account.incomeCount` · skill: backend-developer

### Frontend foundation tests first (see them fail)

- [X] T022 [P] Create `fe/app/lib/ledger.test.ts`: `LEDGER.spendings` reproduces today's labels
  exactly (party "Merchant", "No spendings recorded in October 2026.", "14 spendings", "Add a
  merchant name.", storage keys `hlb.byCategoryView` / `hlb.byAccountView`); `LEDGER.income` gives
  "Source", "No income recorded in October 2026.", "1 entry" / "3 entries", "Add a source.", "Not
  one of your income categories", "Add income", storage keys `hlb.income.byCategoryView` /
  `hlb.income.byAccountView`, sample `/sample-income.csv`, import title "Import income" · skill:
  frontend-developer
- [X] T023 [P] Extend `fe/app/lib/data.test.ts`: `loadLedgerPage('income', '2026-10')` calls
  `/api/v1/monthly-income-totals?year=2026`, `/api/v1/incomes?from=2026-10-01&to=2026-10-31&…`,
  `/api/v1/income-category-totals…`, `/api/v1/income-account-totals…`,
  `/api/v1/income-categories`, `/api/v1/accounts`, `/api/v1/income-sources`, in the same two
  parallel rounds as spendings; incomes map `source` → `merchant` and income categories' and
  sources' `incomeCount` → `count`; `loadSpendingsPage` is unchanged. Extend
  `fe/app/lib/data-earliest.test.ts` for `getEarliestYear('income')` · skill: frontend-developer
- [X] T024 [P] Extend `fe/app/lib/actions.test.ts`: `saveIncome` posts to `/api/v1/incomes` (PUT
  `/api/v1/incomes/{id}` when editing) with `source` (from the UI's `merchant`), an
  `Idempotency-Key`, and maps a `422` at `#/categoryName` to "Not one of your income categories";
  `deleteIncome`; `createIncomeCategory` / `deleteIncomeCategory` with the existing empty and
  duplicate messages; `importIncome` posts `{ incomes: [...] }` to `/api/v1/income-imports` and
  maps row errors at `#/incomes/{i}/{field}` · skill: frontend-developer
- [X] T025 [P] Extend `fe/app/lib/messages.test.ts` (research R-008): `accountUsage(3, 0)` → "3
  spendings recorded" (unchanged), `accountUsage(0, 0)` → "No spendings yet", `accountUsage(3, 1)`
  → "3 spendings · 1 income entry"; account in-use: only spendings → today's wording; with income →
  "Brokerage has 2 spendings and 1 income entry, so it can’t be removed." / "Brokerage has 4 income
  entries, so it can’t be removed."; income category in use → "Freelance is used by 1 entry, so it
  can’t be removed." · skill: frontend-developer

### Frontend foundation implementation

- [X] T026 Add income aliases to `fe/app/lib/definitions.ts` (`Income`, `IncomePage`,
  `IncomeCategory`, `IncomeCategoryList`, `IncomeSourceList`, `IncomeMonthlyTotals`,
  `IncomeCategoryTotals`, `IncomeAccountTotals`, `IncomeImport`), generated types only · skill:
  frontend-developer
- [X] T027 Create `fe/app/lib/ledger.ts` (research R-006): `export type Ledger = 'spendings' |
  'income'` and `LEDGER: Record<Ledger, {...}>` with route (`/spendings`, `/income`), tab label,
  page title, party label, entry noun singular/plural ("spending"/"spendings", "entry"/"entries"),
  empty-month text for desktop and phone, add label, validation texts, not-yours category text,
  phone form titles ("Add spending"/"Add income", "Edit spending"/"Edit income"), import title and
  sample path, and breakdown storage keys per `BreakdownKind`. Values for `'spendings'` are copied
  from today's code so nothing changes · skill: frontend-developer
- [X] T028 Data and actions (research R-007):
  - `fe/app/lib/data.ts`: `loadLedgerPage(ledger, monthParam)` (React `cache`) choosing endpoints
    from a server-side map; `loadSpendingsPage = (m) => loadLedgerPage('spendings', m)`; income
    readers map to the same `LedgerPageData` shape (incomes `source` → `merchant`, totals
    `entryCount` → the `spendingCount` field the components already read, income categories
    `incomeCount` → `count`); `getEarliestYear(ledger)` with `getEarliestSpendingYear` kept as a
    wrapper; accounts gain `incomeCount`;
  - `fe/app/lib/actions.ts`: `saveIncome`, `deleteIncome`, `importIncome` sharing the zod schema
    and error mapping with the spending actions; `createIncomeCategory` / `deleteIncomeCategory`
    in the Settings resources map (`/api/v1/income-categories`). Paths are fixed in server code;
    no client value chooses an endpoint;
  - `fe/app/lib/messages.ts`: the texts from T025.
  · skill: frontend-developer
- [X] T029 Thread `ledger?: Ledger` (default `'spendings'`) through the shared components, reading
  every text and storage key from `LEDGER[ledger]` and calling the ledger's actions:
  `fe/app/ui/spendings/month-section.tsx` (takes `ledger`, calls `loadLedgerPage`),
  `month-line.tsx`, `spendings-table.tsx`, `entry-row.tsx`, `spending-row.tsx`,
  `month-total-card.tsx`, `breakdown-card.tsx` (storage key from `LEDGER[ledger]` × `kind`);
  `fe/app/ui/phone/phone-section.tsx`, `phone-app.tsx`, `spending-list.tsx`, `spending-form.tsx`,
  `delete-sheet.tsx`, `breakdown.tsx`, `month-year-picker.tsx` (first switch option "Spendings" /
  "Income"); `fe/app/ui/import/import-provider.tsx` and the dialogs. Document on the UI entry type
  that `merchant` is "the merchant of a spending, or the source of an income" · skill:
  frontend-developer
- [X] T030 Run `pnpm typecheck && pnpm lint && pnpm test` and `pnpm e2e`: every pre-009 unit and
  e2e test passes unchanged (Spendings must not change) · skill: frontend-developer

**Checkpoint**: the backend serves income; the frontend can load and save it; nothing new is on
screen and Spendings is unchanged.

---

## Phase 3: User Story 1 — See my income for a month (Priority: P1) 🎯 MVP

**Goal**: Spendings | Income tabs on the desktop; the Income tab shows the month line, table,
footer and total card for income.

**Independent Test**: With income in September and October, open the app, switch to Income,
check October's table, footer and card match; click September and check again; switch back to
Spendings on September.

### Tests (write first, see them fail)

- [X] T031 [P] [US1] Extend `fe/app/ui/app-header.test.tsx`: the `Main` nav has links "Spendings"
  and "Income"; the one for the current page has `aria-current="page"`; with month `2026-09` both
  hrefs carry `?month=2026-09` · skill: frontend-developer
- [X] T032 [P] [US1] Extend `fe/app/ui/spendings/spendings-table.test.tsx` and
  `month-line.test.tsx` with `ledger="income"`: columns Date, Source, Category, Account, Amount;
  footer "3 entries" / "1 entry"; empty month "No income recorded in October 2026." · skill:
  frontend-developer
- [X] T033 [P] [US1] Create `fe/e2e/income/month-view.spec.ts` (desktop), seeding through the API
  as `fe/e2e/month-view.spec.ts` does: the app opens on Spendings; select September, click Income →
  URL `/income?month=2026-09`, September selected, no full reload (a `window` marker set before the
  click survives); month line shows income totals only, future months greyed; table, footer and
  card match; empty month message; Spendings → September with spendings only; reload keeps the tab;
  `/` → Spendings · skill: frontend-developer

### Implementation

- [X] T034 [US1] Create `fe/app/income/page.tsx` (twin of `fe/app/spendings/page.tsx`, metadata
  `title: 'Income'`, `robots: { index: false, follow: false }`, rendering `<MonthSection
  ledger="income" …>` and `<PhoneSection ledger="income" …>`), `fe/app/income/loading.tsx` and
  `fe/app/income/error.tsx` (twins). Make sure the Auth0 route protection and the sign-in
  `returnTo` treat `/income` like `/spendings` (check `fe/proxy.ts`, `fe/app/lib/auth0.ts` and
  `fe/app/lib/safe-return-to.ts`; extend `fe/proxy.test.ts` and `safe-return-to.test.ts`) · skill: frontend-developer
- [X] T035 [US1] Create `fe/app/ui/ledger-tabs.tsx`: client component reading the current
  `?month=` and pathname, rendering Next `<Link replace>` items for each `LEDGER` entry with
  `aria-current="page"` on the selected one, in two variants: `nav` (desktop header) and
  `segmented` (phone, used in T052). Use it in `fe/app/ui/app-header.tsx` in place of the single
  Spendings link · skill: frontend-developer
- [X] T036 [US1] Run `pnpm test` and `pnpm e2e --grep income` (desktop) until T031–T033 pass ·
  skill: frontend-developer

**Checkpoint**: income can be viewed by month on the desktop.

---

## Phase 4: User Story 2 — Record, edit and delete income on the desktop (Priority: P1)

**Goal**: The inline entry row, autofill, keyboard entry, edit and one-click remove work on the
Income tab with Source and income categories.

**Independent Test**: With one account, open Income, click +, type "Acme Corp", pick Salary and the
account, type 5000, press Enter; the entry appears in the table, totals and server. Edit its
amount, then remove it.

### Tests (write first, see them fail)

- [X] T037 [P] [US2] Extend `fe/app/ui/spendings/entry-row.test.tsx` with `ledger="income"`:
  placeholder and label "Source"; the category combobox offers only the income categories passed
  in; "Groceries" shows "Not one of your income categories" and blocks save; empty source → "Add a
  source."; accepting a past source fills category and account when empty; saving calls
  `saveIncome` · skill: frontend-developer
- [X] T038 [P] [US2] Create `fe/e2e/income/add-income.spec.ts`, `keyboard-autofill.spec.ts` and
  `edit-delete.spec.ts` (desktop), modelled on the spending specs: + opens the row with the calendar
  (date rules as Spendings); keyboard-only entry with "ac" → "Acme Corp", Tab fills Salary and Debit
  card, Enter saves once; save moves to the entry's month and every figure updates without reload;
  each validation message; unknown account → "Not one of your accounts"; a user with no accounts
  sees the message pointing to Settings; hover → edit and remove; remove deletes at once; switching
  to Spendings with a row open discards it; Spendings figures never change · skill:
  frontend-developer

### Implementation

- [X] T039 [US2] Wire the income entry row: `fe/app/ui/spendings/month-section.tsx` passes income
  categories, accounts and income sources for `ledger="income"`; `entry-row.tsx` and
  `spending-row.tsx` call `saveIncome` / `deleteIncome` and use `LEDGER[ledger]` texts; fix
  anything T037–T038 expose without changing the spending behaviour · skill: frontend-developer
- [X] T040 [US2] Run `pnpm test` and `pnpm e2e --grep income` until T037–T038 pass · skill:
  frontend-developer

**Checkpoint**: US1 + US2 make the desktop Income tab usable (MVP).

---

## Phase 5: User Story 3 — See where the month's income came from (Priority: P2)

**Goal**: By category and By account cards on the Income tab, with fixed colours, slice labels and
their own Table / Pie choices.

**Independent Test**: October income in Salary and Investments on two accounts → By category lists
both with totals equal to the month's income; Pie has two slices in green and blue; By account the
same with the accounts' colours.

### Tests (write first, see them fail)

- [X] T041 [P] [US3] Extend `fe/app/ui/spendings/breakdown-card.test.tsx`: with `ledger="income"`
  each card writes `hlb.income.byCategoryView` / `hlb.income.byAccountView` and never the spending
  keys; Spendings cards never write the income keys; texts "Income by category" summary in the pie
  ("Income by category: 2 categories, largest Salary at 94%") · skill: frontend-developer
- [X] T042 [P] [US3] Create `fe/e2e/income/breakdowns.spec.ts` (desktop): only categories/accounts
  used that month, largest first, totals equal the month's income; Salary slice and swatch
  `#1e5b45`, Investments `#1d4ed8`; an account has the same colour as on Spendings; hover/focus
  label "Salary · $5,000.00 · 94%"; Pie on Income By category while Spendings stays on Table, kept
  after month change, tab switch and reload; empty month message. Add `/income` with both cards (Table
  and Pie) to `fe/e2e/a11y.spec.ts` · skill: frontend-developer

### Implementation

- [X] T043 [US3] In `fe/app/ui/spendings/month-section.tsx` render both `BreakdownCard`s with
  `ledger` and the income totals for `ledger="income"`; in `breakdown-pie.tsx` take the summary noun
  from `LEDGER[ledger]` ("Spending by …" unchanged for spendings, "Income by …" for income) · skill:
  frontend-developer
- [X] T044 [US3] Run `pnpm test` and `pnpm e2e --grep income` until T041–T042 pass · skill:
  frontend-developer

**Checkpoint**: the desktop Income tab is complete apart from Settings and import.

---

## Phase 6: User Story 4 — Manage income categories in Settings (Priority: P2)

**Goal**: Settings shows Spending categories, Income categories and Accounts; income categories
can be added and removed; accounts show both counts and both refusal reasons.

**Independent Test**: Add income category "Freelance"; it appears in Settings and in the Income
dropdown only; record an income in it; removing it is refused; delete the income, then remove it.

### Tests (write first, see them fail)

- [X] T045 [P] [US4] Extend `fe/app/ui/settings/settings-dialog.test.tsx`: three sections in order
  "Spending categories", "Income categories", "Accounts"; Income categories lists Salary and
  Investments marked "Default" with no remove button and custom ones with swatch, count and remove;
  empty and duplicate messages as for categories; in-use refusal "Freelance is used by 1 entry, so
  it can’t be removed."; an account used by income shows "3 spendings · 1 income entry" and the
  refusal with both counts · skill: frontend-developer
- [X] T046 [P] [US4] Extend `fe/app/ui/phone/phone-settings.test.tsx` with the same three sections
  and rules on the phone · skill: frontend-developer
- [X] T047 [P] [US4] Extend `fe/e2e/settings.spec.ts` and `fe/e2e/phone/menu-settings.spec.ts` (done as `fe/e2e/income/settings.spec.ts` and in `fe/e2e/phone/income/income.spec.ts`)
  with the Independent Test above, the "Other" in both lists case, and the account removal refusal
  with income; existing assertions keep passing after the section rename (update only the section
  name lookups) · skill: frontend-developer

### Implementation

- [X] T048 [US4] Create `fe/app/ui/settings/income-categories-panel.tsx` (twin of
  `categories-panel.tsx` on `NamedListPanel`, calling `createIncomeCategory` /
  `deleteIncomeCategory`, swatches, defaults marked); in `settings-dialog.tsx` rename the
  Categories section to "Spending categories" and add Income categories between it and Accounts;
  `settings-section.tsx` loads income categories; `accounts-panel.tsx` shows `accountUsage(s, i)`;
  `fe/app/ui/phone/phone-settings.tsx` gets the same three sections · skill: frontend-developer
- [X] T049 [US4] Run `pnpm test` and `pnpm e2e --grep settings` (desktop and phone) until T045–T047
  pass · skill: frontend-developer

**Checkpoint**: income categories are managed in Settings on desktop and phone.

---

## Phase 7: User Story 6 — Income on the phone (Priority: P2)

**Goal**: The phone header switches Spendings | Income; Income gets the phone list, Add / Edit
income forms, delete sheet and breakdowns.

**Independent Test**: At 390 px wide, switch to Income, tap Add income, fill the form, save; the
entry appears in the list and in By category; delete it through the confirmation sheet.

### Tests (write first, see them fail)

- [X] T050 [P] [US6] Extend `fe/app/ui/phone/phone-app.test.tsx`, `spending-list.test.tsx`,
  `spending-form.test.tsx`, `delete-sheet.test.tsx`, `month-year-picker.test.tsx` and
  `breakdown.test.tsx` with `ledger="income"`: floating button "Add income"; form title "Add
  income" / "Edit income" with a Source field and only income categories in the picker; empty list
  "No income in October 2026."; switch "Income | By category | By account"; income chart choices
  separate from the spending ones · skill: frontend-developer
- [X] T051 [P] [US6] Create `fe/e2e/phone/income/month-view.spec.ts`, `add-edit-delete.spec.ts`
  and `breakdowns.spec.ts` (phone project), modelled on the phone spending specs: header switch
  "Spendings | Income" on one row at 360 and 390 wide, each option at least 44 px tall, no
  horizontal scroll; the Independent Test; top-6 donut; back gesture with a sheet open closes it and
  with nothing open does not switch tabs (FR-072). Add `/income` (list, form, By category) to
  `fe/e2e/phone/a11y.spec.ts` · skill: frontend-developer

### Implementation

- [X] T052 [US6] Render `LedgerTabs variant="segmented"` in `fe/app/ui/phone/phone-header.tsx`
  (or directly under it), keeping `?month=`; in `fe/app/ui/phone/phone-ui-provider.tsx` keep chart
  choices per ledger; wire `phone-section.tsx` / `phone-app.tsx` with `ledger="income"` data and
  actions; fix anything T050–T051 expose without changing the spending phone behaviour · skill:
  frontend-developer
- [X] T053 [US6] Run `pnpm test` and `pnpm e2e --project phone` until T050–T051 pass · skill:
  frontend-developer

**Checkpoint**: income works on desktop and phone.

---

## Phase 8: User Story 5 — Import income from a CSV file (Priority: P3)

**Goal**: Import CSV on the Income tab imports income entries with the same review and fixing.

**Independent Test**: On Income, download the sample, import it, check Salary and Investments rows
are valid, add them, and check they appear on Income and not on Spendings.

### Tests (write first, see them fail)

- [X] T054 [P] [US5] Extend `fe/app/lib/csv-import.test.ts`: a header `Date,Source,Category,
  Account,Amount` maps Source to the party column; `payer` and `employer` too; existing headers
  unchanged · skill: frontend-developer
- [X] T055 [P] [US5] Extend `fe/app/ui/import/import-dialog.test.tsx`, `review-stage.test.tsx` and
  `fe/app/ui/phone/phone-review.test.tsx` with `ledger="income"`: title "Import income"; sample link
  `/sample-income.csv`; a row with "Groceries" flagged "Not one of your income categories", counted
  in "rows need attention", Add greyed out; Add calls `importIncome` · skill: frontend-developer
- [X] T056 [P] [US5] Create `fe/e2e/income/import-csv.spec.ts` and
  `fe/e2e/phone/income/import-csv.spec.ts`, modelled on the spending import specs, covering the
  Independent Test, the flagged row, all-or-nothing on a server refusal, and that importing on
  Spendings still imports spendings · skill: frontend-developer

### Implementation

- [X] T057 [US5] Create `fe/public/sample-income.csv` (header `Date,Source,Category,Account,Amount`;
  rows for Salary and Investments in the current year's format of `sample-spendings.csv`); extend
  the merchant pattern in `fe/app/lib/csv-import.ts` `mapColumns` with `source|payer|employer`
  (research R-009); make the import provider, dialog, review stage and phone import/review use
  `LEDGER[ledger]` texts, income categories for checking, and `importIncome` · skill:
  frontend-developer
- [X] T058 [US5] Run `pnpm test` and `pnpm e2e --grep import` (desktop and phone) until T054–T056
  pass · skill: frontend-developer

**Checkpoint**: all six stories work on desktop and phone.

---

## Phase 9: Polish & Cross-Cutting Concerns

- [X] T059 [P] Update `fe/README.md` and `hl-budget-backend/README.md` where they list screens,
  routes, endpoints or tables · skill: none
- [X] T060 [P] Add one-line notes pointing to 009 in `specs/002-spending-tracking/spec.md`
  (Assumptions: "Spendings only" → "Extended by 009: income"), `specs/003-spending-screens/spec.md`
  (FR-081 Settings: "Changed by 009: Categories is now Spending categories, beside Income
  categories") and `specs/006-mobile-view/spec.md` (FR-017) · skill: none
- [X] T061 Run the full gates: `./gradlew check` in the backend; `pnpm typecheck && pnpm lint &&
  pnpm test && pnpm e2e` in the frontend (desktop and phone projects, axe) · skill: none
- [X] T062 Backend release first: open the PR, merge after the user agrees, let the release
  workflow open the hl-helm PR, merge it after the user agrees, then verify with
  `homelab:hl-deploy-status` (V7 applied, pod Ready, new endpoints answer `401` without a token) ·
  skill: homelab:hl-deploy-status
- [X] T063 Frontend release: the same flow, then `homelab:hl-deploy-status` · skill:
  homelab:hl-deploy-status
- [ ] T064 Live check on https://budget.valyalkin.com on a desktop and a phone (quickstart §5):
  the existing user sees Salary and Investments, records one income on each device, Spendings
  figures unchanged. Then update the README Features table in the spec repo with a small PR (row
  for 009, "Implemented <date>") · skill: none

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (T001)** → everything.
- **Foundational**:
  - Backend tests T002–T013 can all be written right after T001.
  - Backend implementation: T014 → T015 → T016 → T017 → T018, T019, T020 (T018–T020 in any order
    after T017; T019 also needs T017's `IncomeEntity`) → T021.
  - Frontend tests T022–T025 can be written right after T001; T026 needs T021's regenerated
    types; T026 → T027 → T028 → T029 → T030.
- **US1** (T031–T036): needs Phase 2.
- **US2** (T037–T040): needs US1's route (T034).
- **US3** (T041–T044): needs US1's route; independent of US2.
- **US4** (T045–T049): needs Phase 2 only (Settings is reachable from Spendings); independent of
  US1–US3. Its income-dropdown check in T047 needs US2.
- **US6** (T050–T053): needs Phase 2 and T034–T035 (route and tabs component).
- **US5** (T054–T058): needs US1 (desktop) and US6 (phone import entry point).
- **Polish**: after the stories you intend to ship.

### User Story Dependencies

- US1 (view) — Phase 2. MVP with US2.
- US2 (add/edit/delete) — US1.
- US3 (breakdowns) — US1.
- US4 (Settings) — Phase 2.
- US6 (phone) — US1's route and tabs.
- US5 (import) — US1 and US6.

### Within Each User Story

Tests first and failing → wiring → run the story's tests → checkpoint.

### Parallel Opportunities

- T002–T013 together (different test files), and T022–T025 alongside them.
- T018, T019 and T020 together after T017.
- After US1: US2, US3, US4 and US6 can proceed in parallel (different files, except shared edits to
  `month-section.tsx` in US2 and US3 — do those one after the other).
- Within stories: T031–T033; T037–T038; T041–T042; T045–T047; T050–T051; T054–T056.
- T059 with T060.

---

## Parallel Example: Foundational backend tests

```text
Task: "T002 IncomeMigrationIT in …/test/repository/IncomeMigrationIT.kt"
Task: "T003 DefaultIncomeCategoriesIT in …/test/controller/DefaultIncomeCategoriesIT.kt"
Task: "T004 IncomeCategoryControllerIT in …/test/controller/IncomeCategoryControllerIT.kt"
Task: "T006 IncomeControllerIT in …/test/controller/IncomeControllerIT.kt"
Task: "T009 IncomeTotalsIT in …/test/controller/IncomeTotalsIT.kt"
Task: "T011 IncomeImportsIT in …/test/controller/IncomeImportsIT.kt"
```

## Parallel Example: tracks after US1

```text
Developer A (desktop): T037, T038 → T039 → T040 → T041, T042 → T043 → T044
Developer B (settings): T045–T047 → T048 → T049
Developer C (phone):    T050, T051 → T052 → T053 → T054–T058
```

---

## Implementation Strategy

### MVP First (User Story 1 + 2)

1. Phase 1, then Phase 2 (backend can be released on its own: the changes are additive).
2. US1: the Income tab and month view on the desktop.
3. US2: recording, editing and deleting income on the desktop.
4. **Stop and validate**: quickstart §3 and §4.1–4.4. Release if wanted (Settings still lacks
   income categories, but Salary and Investments cover the common case).

### Incremental Delivery

1. Phase 2 → backend release (nothing visible changes).
2. US1 + US2 → Income tab on the desktop.
3. US3 → income breakdowns.
4. US4 → income categories in Settings, account counts.
5. US6 → income on the phone.
6. US5 → income CSV import.
7. Polish → docs, gates, frontend release, live check, README.
