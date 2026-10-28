---

description: "Task list for 002-spending-tracking"
---

# Tasks: Spending Tracking (Backend)

**Input**: Design documents from `/specs/002-spending-tracking/`

**Prerequisites**: plan.md, spec.md, research.md, data-model.md, contracts/openapi.yaml,
quickstart.md

**Tests**: REQUIRED. Constitution Principle IV ("tests ship with the code") and research R-013
map every success criterion to a test. Within each story, write the tests first and see them
fail before writing the code that satisfies them.

**Organization**: Tasks are grouped by user story (spec.md US1–US5), so each story can be built
and checked on its own.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Can run in parallel (different files, no dependency on an unfinished task)
- **[Story]**: The user story the task belongs to (US1 … US5)
- Every task ends with `· skill: <name>`. Load that skill before starting the task; its
  definition of done is part of the task's done (plan.md "Skills that drive the
  implementation"). Where a skill and the constitution disagree, the constitution wins
  (research R-006, R-009).

## Path Conventions

All paths are relative to the spec repo root `/Users/valyalkin/Documents/Projects/hl-budget-spec/`
unless they start with `~`.

- `hl-budget-backend/`: the git worktree of `valyalkin/hl-budget-backend`, on branch
  `002-spending-tracking` (T002).
  - Main code: `hl-budget-backend/src/main/kotlin/com/valyalkin/hlbudget/`, written below as
    `…/main/`.
  - Tests: `hl-budget-backend/src/test/kotlin/com/valyalkin/hlbudget/`, written below as
    `…/test/`.
- `hl-budget-frontend/`: the git worktree of `valyalkin/hl-budget-frontend`.
- `~/Documents/Projects/hl-helm/`: the homelab chart repo.
- **Commits**: commit inside the service repos (not this spec repo, which ignores them), using
  Conventional Commits, after each task or logical group.
- **Merging**: merging to `main` in a service repo triggers a release. Do it only after the gate
  is green and the user agrees.

Shared rules that several tasks reference:

- **Name rule** (accounts, categories; FR-013, FR-022, R-004): trim, then "must not be blank",
  max **50** characters, unique per user **ignoring case**.
  - DB: `CHECK (length(name) BETWEEN 1 AND 50 AND name = btrim(name))` and a unique index on
    `(user_id, lower(name))`.
  - Duplicates: a service pre-check (excluding the item itself on rename) plus catching
    `DataIntegrityViolationException`. Both → `DuplicateException` 409 with
    `details = {reason: "duplicate-name", name}`.
- **Not yours = not found** (FR-002): every lookup filters by the caller's `userId`. A path id
  that is missing or another user's → `NotFoundException` 404 with
  `details = {resource, <resource>Id}`. Both cases give the same response.

---

## Phase 1: Setup

**Purpose**: The backend worktree is on a fresh feature branch with the new configuration
documented.

- [X] T001 Operator step (ask the user, do not do it yourself): `hl-budget-backend/` is on branch `fix/api-docs-off-in-cluster` (commit `945c3a8`). Ask whether to merge it to `main` first or park it, and wait for the answer (research R-001) · skill: none
- [X] T002 In `hl-budget-backend/`, run `git fetch origin` and `git switch -c 002-spending-tracking origin/main`. Confirm `./gradlew check` is green before any change · skill: backend-developer
- [X] T003 [P] Add `HLBUDGET_MONEY_CURRENCY=CAD` and `HLBUDGET_TIMEZONE=America/Toronto`, each with a one-line comment ("ISO 4217; every amount is in this currency; changing it later does not convert data" / "IANA zone used to decide what 'today' is"), to `hl-budget-backend/.env.example` and to the developer's untracked `hl-budget-backend/.env.local` · skill: backend-developer

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: Configuration, error handling, schema, entities and caller resolution. Every story
needs these.

**⚠️ CRITICAL**: No user-story work starts until this phase is complete.

### Tests for the foundation ⚠️ (write first, see them fail)

- [X] T004 [P] Extend `…/test/config/HlBudgetPropertiesTest.kt`. A missing or blank `hlbudget.money.currency` fails binding, naming the property. So does a code that is not a valid ISO 4217 code (`XYZ`, `cad`). A missing or invalid `hlbudget.time-zone` (`Mars/Base`) also fails. `CAD` and `America/Toronto` bind to `java.util.Currency` and `java.time.ZoneId` · skill: backend-developer
- [X] T005 [P] Extend `…/test/error/GlobalExceptionHandlerTest.kt` and `…/test/error/GlobalExceptionHandlerIT.kt` (with test endpoints in `hl-budget-backend/src/test/kotlin/com/valyalkin/failures/FailureController.kt`):
  - **422 cases**: these give 422 `urn:problem:hl-budget:unprocessable-entity` with an `errors` array of `{pointer, field, code, message}`:
    - a bean-validation failure in a body, a query parameter or a header
    - a missing body field (`Required`)
    - a wrong JSON type for a body field (`TypeMismatch`)
    - an unparsable query parameter or header
  - **400 cases**: a body that is not JSON, and a path variable that is not a UUID, give 400 `bad-request` with no `errors`.
  - **New exceptions**: `ValidationException(listOf(FieldViolation(...)))` renders 422 with those `errors`. `ConflictException("…", mapOf("reason" to "in-use"))` renders 409 `conflict` with `details.reason`.
  - **Privacy**: no rejected value is echoed anywhere (research R-009) · skill: backend-developer
- [X] T006 [P] Extend `…/test/repository/SchemaValidationIT.kt` against Flyway V1–V3:
  - **Tables**: `users.categories_seeded_at` exists and is nullable. `accounts`, `categories` and `spendings` have exactly the columns, constraints and indexes in data-model.md.
  - **Name constraint**: inserting a name of `' x'` or one of 51 characters fails `ck_*_name`.
  - **Uniqueness**: a second `lower(name)` for the same user fails `ux_*_user_lower_name`, but the same name for another user succeeds.
  - **Amount**: a spending with `amount = 0` fails `ck_spendings_amount`.
  - **Cross-user reference**: a spending whose `account_id` belongs to another user fails the composite FK.
  - **Delete refusal**: deleting an account or category that a spending uses fails the FK.
  - **Idempotency pair**: a key without a fingerprint fails `ck_spendings_idempotency_pair`.
  - **Key uniqueness**: two spendings with the same `(user_id, idempotency_key)` fail, but two with null keys succeed · skill: backend-developer
- [X] T007 [P] Write `…/test/service/CurrentUserServiceTest.kt` (MockK). `resolve(identity)` returns the id of an existing row with one lookup. When the row is missing it inserts with "do nothing on conflict", looks up again and returns that id. It never touches `display_name`, `email` or `avatar_url` (research R-002) · skill: backend-developer

### Foundation implementation

- [X] T008 Extend `…/main/config/HlBudgetProperties.kt` with `money: Money(currency: String)` and `timeZone: String`. Both are `@field:NotBlank` with empty defaults. Add custom validation (or a validated `init` exposing `currency: java.util.Currency` and `zoneId: java.time.ZoneId`) so an invalid value fails startup naming the property. Bind them in `hl-budget-backend/src/main/resources/application.yaml` as `hlbudget.money.currency: ${HLBUDGET_MONEY_CURRENCY:}` and `hlbudget.time-zone: ${HLBUDGET_TIMEZONE:}`, required with **no default**, plus a comment like the `auth` block's. Set `CAD` and `America/Toronto` in `hl-budget-backend/src/main/resources/application-local.yaml` and `hl-budget-backend/src/test/resources/application-test.yaml` (research R-006) · skill: backend-developer
- [X] T009 [P] Create `…/main/config/ClockConfig.kt`: a `@Bean fun clock(): Clock = Clock.systemUTC()`. In `…/test/support/`, add a way to replace it with a fixed clock in ITs (for example a `@TestConfiguration` with `@Primary` `Clock.fixed(Instant.parse("2026-10-27T15:00:00Z"), UTC)`) · skill: backend-developer
- [X] T010 Change the error package (research R-009):
  - In `…/main/error/BusinessException.kt`, add `class ConflictException(message, details = emptyMap(), cause = null)` (409) and `class ValidationException(val violations: List<FieldViolation>, message: String = …)`, which extends `UnprocessableEntityException` (422). Its `detail` reads "<field>: <message>" for one violation and "N fields are invalid" for several.
  - In `…/main/error/GlobalExceptionHandler.kt`, switch `validationProblem` to 422. Route these to it: `MethodArgumentNotValidException`, `HandlerMethodValidationException`, `ConstraintViolationException`, `MissingServletRequestParameterException`, `MissingRequestHeaderException`, `MethodArgumentTypeMismatchException` on query or header parameters, and `HttpMessageNotReadableException` caused by a missing or mistyped field (`Required`/`TypeMismatch` with a pointer).
  - Keep 400 for unreadable JSON and for `MethodArgumentTypeMismatchException` on a **path** variable.
  - Render `ValidationException.violations` as `errors`.
  - Make T005 green · skill: backend-developer
- [X] T011 Create `hl-budget-backend/src/main/resources/db/migration/V2__add_users_categories_seeded_at.sql`: `ALTER TABLE users ADD COLUMN categories_seeded_at timestamptz NULL;`, with a comment pointing to research R-003 · skill: backend-developer
- [X] T012 Create `hl-budget-backend/src/main/resources/db/migration/V3__create_accounts_categories_spendings.sql` exactly per data-model.md:
  - **`accounts`**:
    - Columns: `id uuid NOT NULL DEFAULT gen_random_uuid()`, `user_id uuid NOT NULL`, `name varchar(50) NOT NULL`, `created_at timestamptz NOT NULL DEFAULT now()`, `updated_at timestamptz NOT NULL DEFAULT now()`, `version bigint NOT NULL DEFAULT 0`.
    - Constraints and index: `pk_accounts`, `uq_accounts_id_user UNIQUE (id, user_id)`, `fk_accounts_user` → `users(id) ON DELETE CASCADE`, `ck_accounts_name CHECK (length(name) BETWEEN 1 AND 50 AND name = btrim(name))`, `CREATE UNIQUE INDEX ux_accounts_user_lower_name ON accounts (user_id, lower(name))`.
  - **`categories`**: the same shape plus `is_default boolean NOT NULL DEFAULT false` (`pk_categories`, `uq_categories_id_user`, `fk_categories_user`, `ck_categories_name`, `ux_categories_user_lower_name`).
  - **`spendings`**:
    - Columns: `id uuid NOT NULL DEFAULT gen_random_uuid()`, `user_id uuid NOT NULL` (FK `fk_spendings_user` → `users(id) ON DELETE CASCADE`), `spending_date date NOT NULL`, `merchant varchar(255) NOT NULL`, `amount numeric(10,2) NOT NULL`, `category_id uuid NOT NULL`, `account_id uuid NOT NULL`, `idempotency_key varchar(255) NULL`, `idempotency_fingerprint char(64) NULL`, `created_at timestamptz NOT NULL DEFAULT now()`, `updated_at timestamptz NOT NULL DEFAULT now()`, `version bigint NOT NULL DEFAULT 0`.
    - Constraints: `pk_spendings`; `fk_spendings_category FOREIGN KEY (category_id, user_id) REFERENCES categories (id, user_id) ON DELETE NO ACTION`; `fk_spendings_account FOREIGN KEY (account_id, user_id) REFERENCES accounts (id, user_id) ON DELETE NO ACTION`; `ck_spendings_amount CHECK (amount > 0)`; `ck_spendings_merchant CHECK (length(merchant) BETWEEN 1 AND 255 AND merchant = btrim(merchant))`; `ck_spendings_idempotency_pair CHECK ((idempotency_key IS NULL) = (idempotency_fingerprint IS NULL))`.
    - Indexes: `ux_spendings_user_idempotency_key UNIQUE (user_id, idempotency_key) WHERE idempotency_key IS NOT NULL`, `ix_spendings_user_date (user_id, spending_date DESC, created_at DESC, id)`, `ix_spendings_user_category (user_id, category_id)`, `ix_spendings_user_account (user_id, account_id)` · skill: backend-developer
- [X] T013 Update and create the entities (`class`, `var`, every column named, `@Version`):
  - `…/main/repository/UserEntity.kt`: add `categoriesSeededAt: Instant?`. The 001 upsert must not touch it.
  - `…/main/repository/account/AccountEntity.kt`: on `accounts`.
  - `…/main/repository/category/CategoryEntity.kt`: on `categories`, with `isDefault`.
  - `…/main/repository/spending/SpendingEntity.kt`: on `spendings`. `amount: BigDecimal` with `precision = 10, scale = 2`; `date` mapped to column `spending_date`; `categoryId` and `accountId` as plain `UUID` columns (no `@ManyToOne`, research R-010); the idempotency columns.
  - Make T006 green · skill: backend-developer
- [X] T014 Create `…/main/repository/CurrentUserRepository.kt` (`JdbcClient`) with:
  - `findBySubject(sub): CurrentUserRow(id, categoriesSeededAt)?`
  - `insertIfMissing(sub)`: `INSERT INTO users (auth0_subject) VALUES (:sub) ON CONFLICT (auth0_subject) DO NOTHING`
  - `claimSeeding(userId): Boolean`: `UPDATE users SET categories_seeded_at = now() WHERE id = :id AND categories_seeded_at IS NULL RETURNING id`

  Then create `…/main/service/CurrentUserService.kt` with `@Transactional fun resolve(identity: VerifiedIdentity): UUID` (find → insert-if-missing → find; seeding is added in T020). Make T007 green · skill: backend-developer
- [X] T015 [P] Create the shared request type `…/main/dto/NameRequest.kt`: `data class NameRequest(@field:NotNull(message = "is required") @field:NotBlank(message = "must not be blank") @field:Size(max = 50, message = "must be at most 50 characters") @field:Schema(description = "Trimmed; unique per user ignoring case", example = "Amex") val name: String)`. Add `fun NameRequest.toCommand() = NameCommand(name.trim())`, and create `…/main/model/NameCommand.kt`. Size is checked again after trimming in the services · skill: backend-developer
- [X] T016 [P] Extend `…/main/config/OpenApiConfig.kt`:
  - Tags `Accounts` ("Where the caller's money was paid from"), `Categories` ("What the caller's money was spent on, including the 17 defaults") and `Spendings` ("The caller's spending transactions").
  - Reusable components with the exact examples from `specs/002-spending-tracking/contracts/openapi.yaml`: header `Location`; responses `BadRequest`, `NotFound`, `DuplicateName`, `DuplicateNameOrConcurrent`, `ConcurrentModification`, `InUse`, `DefaultCategory`, `ValidationFailed`, `ValidationOrDefaultCategory`, `SpendingValidationFailed` · skill: backend-developer
- [X] T017 [P] Extend `…/test/support/` (`AbstractIntegrationTest.kt`, `JwtTestTokens.kt`):
  - Tokens for two distinct users (`userA`, `userB`) and a fresh random `sub` per test.
  - Helpers `createAccount(token, name)`, `listCategories(token)` and `createSpending(token, …)` that call the API.
  - A `jdbc` helper for direct row counts.
  - The fixed `Clock` from T009 · skill: backend-developer

**Checkpoint**: `./gradlew check` is green. V2 and V3 apply, the context starts with
`ddl-auto: validate`, and the 001 tests still pass, with the 400→422 changes made only where
T005 says.

---

## Phase 3: User Story 1 — Start with ready-made categories (Priority: P1) 🎯 MVP

**Goal**: The first budgeting call of any user (new or pre-existing) gives them exactly the 17
defaults, once, with no setup.

**Independent Test**: `GET /api/v1/categories` as a new user, as a second new user, and as a user
whose `users` row predates V2 (`categories_seeded_at` null) returns exactly 17 defaults each.
Ten parallel first calls still give 17 (SC-002).

### Tests for User Story 1 ⚠️

- [X] T018 [P] [US1] Extend `…/test/service/CurrentUserServiceTest.kt`. When `categoriesSeededAt` is null and `claimSeeding` returns true, the 17 `DefaultCategories.NAMES` are inserted for that user with `isDefault = true`. When `claimSeeding` returns false, nothing is inserted. When `categoriesSeededAt` is set, `claimSeeding` is not called · skill: backend-developer
- [X] T019 [P] [US1] Write `…/test/controller/DefaultCategoriesIT.kt`:
  - **New user**: `GET /api/v1/categories` gives 200 and `items` with exactly the names `Rent, Groceries, Public Transport, Car, Household, Health, Beauty, Clothes, Education, Travel, Bills, Subscriptions, Other, Sport, Entertainment, Pet, Taxi`, sorted case-insensitively by name. Each item has `isDefault: true`, `spendingCount: 0` and a UUID `id`.
  - **Repeat call**: still 17.
  - **Pre-existing user**: a user created through `PUT /api/v1/users/me`, then with `categories_seeded_at` set to null by SQL and their categories removed, gets 17.
  - **Parallel first calls**: 10 parallel first calls for one new `sub` all return 200, and `select count(*) from categories where user_id = ?` is 17.
  - **Isolation**: user A and user B each get their own 17 ids, with no overlap.
  - **Fetch one**: `GET /api/v1/categories/{id}` for one of them gives 200 with the same shape. Another user's id, or an unknown UUID, gives 404 `not-found`; `/categories/abc` gives 400.
  - **Auth**: no token gives 401 · skill: backend-developer

### Implementation for User Story 1

- [X] T020 [US1] Create `…/main/model/DefaultCategories.kt` (`val NAMES: List<String>`, the 17 names exactly as in T019, in that order). In `CurrentUserService.resolve`, when `categoriesSeededAt == null` and `claimSeeding(userId)` returns true, insert one `categories` row per name with `is_default = true` in the same transaction. Use a batch insert in `…/main/repository/category/CategoryRepository.kt` or `CurrentUserRepository`. Make T018 green · skill: backend-developer
- [X] T021 [P] [US1] Create `…/main/model/Category.kt` (`id: UUID`, `name: String`, `isDefault: Boolean`, `spendingCount: Long`; immutable, no annotations). Create `…/main/repository/category/CategoryRepository.kt` (`JpaRepository<CategoryEntity, UUID>`) with:
  - `findAllWithCounts(userId)`: one JPQL query, `LEFT JOIN` to `SELECT s.categoryId, count(s) FROM SpendingEntity s WHERE s.userId = :userId GROUP BY s.categoryId`, ordered by `lower(name)`, then `id` (research R-011).
  - `findWithCount(userId, id)`.

  Create `…/main/repository/category/CategoryMappings.kt` with `toModel()` · skill: backend-developer
- [X] T022 [US1] Create `…/main/service/CategoryService.kt` (`@Transactional(readOnly = true)` class) with `list(identity): List<Category>` and `get(identity, id): Category`. Both call `currentUser.resolve(identity)` first. `get` throws `NotFoundException("Category $id was not found", mapOf("resource" to "category", "categoryId" to id))` when the category is missing or belongs to someone else. Because `resolve` may write (seeding), mark these methods `@Transactional` (read-write) · skill: backend-developer
- [X] T023 [P] [US1] Create `…/main/dto/category/CategoryResponse.kt` (`id`, `name`, `isDefault`, `spendingCount`, with `@Schema` and `readOnly` on `isDefault`/`spendingCount`), `…/main/dto/category/CategoryListResponse.kt` (`items: List<CategoryResponse>`) and `…/main/dto/category/CategoryMappings.kt` (`Category.toResponse()`) · skill: backend-developer
- [X] T024 [US1] Create `…/main/controller/CategoryController.kt` at `/api/v1/categories`, tag `Categories`:
  - `GET ""`: `operationId = "listCategories"`, responses 200 (`CategoryList` example), 401, 500, 503.
  - `GET "/{categoryId}"`: `operationId = "getCategory"`, responses 200, 400, 401, 404, 500, 503.
  - Both take a `VerifiedIdentity` parameter (hidden in the docs). Make T019 green · skill: backend-developer

**Checkpoint**: US1 works on its own. Every user sees 17 defaults, created exactly once.

---

## Phase 4: User Story 2 — Manage my accounts (Priority: P1)

**Goal**: Create, list, fetch, rename and delete accounts (name only), with case-insensitive
unique names, a spending count, and delete refused while spendings use the account.

**Independent Test**: Create "Amex" and "Cash", list them (sorted, both `spendingCount: 0`),
rename "Amex" to "Amex Gold", delete "Cash", and check that the list reflects each step. User B
cannot see or change them.

### Tests for User Story 2 ⚠️

- [X] T025 [P] [US2] Write `…/test/service/AccountServiceTest.kt` (MockK):
  - **Create**: trims the name. A duplicate found by the pre-check, or a `DataIntegrityViolationException` on save, gives `DuplicateException` with `reason = "duplicate-name"`.
  - **Rename**: renaming to the same name, or to a case-only change, is allowed, because the pre-check excludes the item itself.
  - **Not found**: rename or delete of a missing or foreign id gives `NotFoundException`.
  - **Delete in use**: with a spending count above 0, delete gives `ConflictException` with `details = {reason: "in-use", spendingCount: n}` and nothing is deleted. A `DataIntegrityViolationException` on delete is re-counted and gives the same exception.
  - **Delete unused**: with a count of 0, the row is deleted · skill: backend-developer
- [X] T026 [P] [US2] Write `…/test/controller/AccountControllerIT.kt`:
  - **Create**: `POST /api/v1/accounts {"name":"Amex"}` gives 201, `Location: /api/v1/accounts/{id}` and `{id, name: "Amex", spendingCount: 0}`.
  - **Name validation**: `{"name":"  "}`, `{}` and a 51-character name each give 422 with one `errors` entry at `#/name` (`NotBlank`/`Required`/`Size`). A body that is not JSON gives 400.
  - **Duplicate**: `{"name":" amex "}` gives 409 with `details.reason: duplicate-name`.
  - **List**: `GET` gives `items` sorted by name.
  - **Fetch one**: `GET /{id}` gives 200.
  - **Rename**: `PUT /{id} {"name":"Amex Gold"}` gives 200 with the new name; `PUT` to `"AMEX GOLD"` gives 200; `PUT` to the name of another account gives 409.
  - **Delete**: `DELETE /{id}` gives 204, and a second `DELETE` gives 404.
  - **Other users**: user B's `GET`, `PUT` and `DELETE` on user A's id give 404, and A's account is unchanged.
  - **New user**: user A starts with `items: []` (FR-023).
  - **Auth**: no token gives 401 · skill: backend-developer

### Implementation for User Story 2

- [X] T027 [P] [US2] Create `…/main/model/Account.kt` (`id`, `name`, `spendingCount: Long`). Create `…/main/repository/account/AccountRepository.kt` (`JpaRepository<AccountEntity, UUID>`) with:
  - `findAllWithCounts(userId)` and `findWithCount(userId, id)`, the same shape as T021 (counts via `SpendingEntity.accountId`, ordered `lower(name)`, `id`).
  - `findByIdAndUserId`.
  - `existsByUserIdAndNameIgnoreCase(userId, name)` and `existsByUserIdAndNameIgnoreCaseAndIdNot(userId, name, id)`.

  Also create `…/main/repository/spending/SpendingRepository.kt` with `countByUserIdAndAccountId` and `countByUserIdAndCategoryId` (the rest of it comes in US3/US4). Create `…/main/repository/account/AccountMappings.kt` · skill: backend-developer
- [X] T028 [US2] Create `…/main/service/AccountService.kt` (`@Transactional(readOnly = true)` class, `@Transactional` on writes, each method starting with `currentUser.resolve(identity)`): `list`, `get`, `create(NameCommand)`, `rename(id, NameCommand)` and `delete(id)`.
  - Apply the **Name rule** (Path Conventions) and **Not yours = not found** (resource `account`).
  - `rename` sets `updatedAt = Instant.now(clock)`.
  - `delete` counts first. If the count is above 0, throw `ConflictException("Account $id is used by $n spendings", mapOf("reason" to "in-use", "spendingCount" to n))`. Wrap `deleteById` plus `flush()` and catch `DataIntegrityViolationException`, then re-count and throw the same (research R-005).
  - Make T025 green · skill: backend-developer
- [X] T029 [P] [US2] Create `…/main/dto/account/AccountResponse.kt` (`id`, `name`, `spendingCount` read-only), `…/main/dto/account/AccountListResponse.kt` (`items`) and `…/main/dto/account/AccountMappings.kt` · skill: backend-developer
- [X] T030 [US2] Create `…/main/controller/AccountController.kt` at `/api/v1/accounts`, tag `Accounts`, with `@Valid @RequestBody NameRequest` and responses exactly as in `contracts/openapi.yaml`:
  - `listAccounts`: 200, 401, 500, 503.
  - `createAccount`: `ResponseEntity` 201 + `Location`; 400, 401, 409, 422, 500, 503.
  - `getAccount`: 200, 400, 401, 404, 500, 503.
  - `renameAccount`: 200, 400, 401, 404, 409, 422, 500, 503.
  - `deleteAccount`: `@ResponseStatus(NO_CONTENT)`; 204, 400, 401, 404, 409, 500, 503.

  Make T026 green · skill: backend-developer

**Checkpoint**: US1 and US2 both work on their own. A user has categories and can create the
account they need for a spending.

---

## Phase 5: User Story 3 — Record a spending (Priority: P1)

**Goal**: Create, fetch, update (full `PUT`) and delete a spending. Money is exact and in the
configured currency, the date is not in the future, references are the caller's own, and create
is idempotent.

**Independent Test**: With one account and the defaults, create a spending (201), fetch it,
`PUT` a new amount and category, then delete it, checking the stored values after each step. The
same create sent twice with one `Idempotency-Key` gives one row.

### Tests for User Story 3 ⚠️

- [X] T031 [P] [US3] Write `…/test/service/SpendingRulesTest.kt` (no Spring, fixed `Clock` at `2026-10-28T03:30:00Z`, zone `America/Toronto`, so "today" is `2026-10-27` locally while UTC is already the 28th):
  - **Date**: `2026-10-27` is accepted. `2026-10-28` is a violation at `#/date` with code `NotFuture` and message "must be today or earlier".
  - **Amount**: `"0"` and `"0.00"` are violations at `#/amount/amount` with code `Positive`.
  - **Currency**: `EUR` is a violation at `#/amount/currency` with code `UnsupportedCurrency` and message "must be CAD".
  - **Fingerprint**: equal for commands that differ only in merchant surrounding spaces and in `"84.3"` vs `"84.30"`; different when any field differs; 64 lowercase hex characters (research R-006, R-007) · skill: backend-developer
- [X] T032 [P] [US3] Write `…/test/service/SpendingServiceTest.kt` (MockK):
  - **References**: an unknown or foreign `categoryId`/`accountId` gives `ValidationException` with a `NotFound` violation at `#/categoryId` or `#/accountId`, and all violations (date, amount, currency, references) are reported together in one exception.
  - **Create with key**:
    - Existing key, same fingerprint → returns `replayed = true` and the stored spending.
    - Existing key, different fingerprint → `UnprocessableEntityException` with `reason = "idempotency-key-reused"`.
    - Insert loses the unique race (`DataIntegrityViolationException`) → reloads by key and applies the same rules.
  - **Update**: sets `updatedAt` from the clock, keeps `createdAt` and the idempotency columns, and a missing or foreign id gives `NotFoundException` (resource `spending`).
  - **Delete**: a missing id gives `NotFoundException` · skill: backend-developer
- [X] T033 [P] [US3] Write `…/test/controller/SpendingControllerIT.kt`:
  - **Create**: `POST /api/v1/spendings` with `{"date":"2026-10-20","merchant":"Loblaws","amount":{"amount":"84.37","currency":"CAD"},"categoryId":<Groceries>,"accountId":<Amex>}` gives 201, `Location: /api/v1/spendings/{id}` and a body matching the `SpendingLoblaws` example shape:
    - `amount.amount` is the string `"84.37"`
    - `category {id, name: "Groceries"}`
    - `account {id, name: "Amex"}`
    - `createdAt == updatedAt`, ISO-8601 UTC with `Z`
  - **Scale**: `"84.3"` is returned as `"84.30"`.
  - **Fetch one**: `GET /{id}` gives the same body.
  - **Update**: `PUT /{id}` with a new amount and category gives 200, changed fields, the same `createdAt` and a later `updatedAt`.
  - **Delete**: `DELETE /{id}` gives 204; `GET` then gives 404.
  - **Validation**: one request with a missing `merchant`, `"amount": 84.37` (a JSON number), `"amount":"1.234"` and a future date gives 422 with an `errors` entry for each (`Required`, `TypeMismatch`, `Pattern`, `NotFuture`). A 256-character merchant gives `Size`. `"99999999.99"` is accepted and `"100000000.00"` gives `Pattern`.
  - **Other users' references**: user B's account id in user A's request gives 422 `NotFound` at `#/accountId`.
  - **Other users' paths**: user B's `GET`, `PUT` and `DELETE` on user A's spending give 404 · skill: backend-developer
- [X] T034 [P] [US3] Write `…/test/controller/IdempotencyIT.kt` (SC-007):
  - **Replay**: two sequential `POST`s with `Idempotency-Key: k1` and the same body give 201, then 200 with `Idempotent-Replayed: true`, the same `id` and the same `Location`.
  - **Parallel**: 10 parallel identical `POST`s with key `k2` give exactly one 201, nine 200s and one row in the database.
  - **Reused key**: `k1` with a different merchant gives 422 `details.reason: idempotency-key-reused`.
  - **Per user**: the same key used by user B creates B's own spending.
  - **Bad keys**: an empty key and a 256-character key give 422 at the header.
  - **No key**: two `POST`s without a key create two rows.
  - **Key freed**: after `DELETE`, the key `k1` creates a new spending · skill: backend-developer
- [X] T035 [P] [US3] Write `…/test/controller/DeleteInUseIT.kt` (accounts part):
  - With 2 spendings on "Amex", `DELETE /api/v1/accounts/{amex}` gives 409 with `details: {reason: "in-use", spendingCount: 2}`, the account still exists, and `GET /accounts` shows `spendingCount: 2`.
  - After `PUT`ting both spendings to "Cash", the delete gives 204 · skill: backend-developer

### Implementation for User Story 3

- [X] T036 [P] [US3] Create `…/main/model/Spending.kt` with `Spending(id, date: LocalDate, merchant, amount: Money, category: Ref, account: Ref, createdAt: Instant, updatedAt: Instant)`, `Money(amount: BigDecimal, currency: Currency)` and `Ref(id: UUID, name: String)`. Create `…/main/model/SpendingCommand.kt` with `SpendingCommand(date, merchant, amount: BigDecimal, currency: String, categoryId, accountId)` and `CreateResult(spending, replayed: Boolean)` · skill: backend-developer
- [X] T037 [P] [US3] Create `…/main/dto/MoneyDto.kt`: `data class MoneyDto(@field:NotNull(message = "is required") @field:Pattern(regexp = "^\\d{1,8}(\\.\\d{1,2})?$", message = "must be a decimal string with up to 8 digits and 2 decimals") val amount: String, @field:NotNull(message = "is required") @field:Pattern(regexp = "^[A-Z]{3}$", message = "must be a 3-letter ISO 4217 code") val currency: String)`. Add `Money.toDto()`, which renders `amount.setScale(2).toPlainString()`. Also create `…/main/dto/RefDto.kt` · skill: backend-developer
- [X] T038 [P] [US3] Create `…/main/dto/spending/SpendingRequest.kt`: `date: LocalDate` (`@field:NotNull`), `merchant: String` (`@field:NotBlank`, `@field:Size(max = 255)`), `@field:Valid amount: MoneyDto`, `categoryId: UUID` and `accountId: UUID` (`@field:NotNull`), with `@Schema` examples from the contract. Add `toCommand()`, which trims the merchant and parses the amount with `BigDecimal(amount)`. Also create `…/main/dto/spending/SpendingResponse.kt` (`id, date, merchant, amount: MoneyDto, category: RefDto, account: RefDto, createdAt, updatedAt`) and `…/main/dto/spending/SpendingMappings.kt` · skill: backend-developer
- [X] T039 [US3] Create `…/main/service/SpendingRules.kt` (a `@Component` with `Clock` and `HlBudgetProperties`):
  - `violations(command): List<FieldViolation>` for `NotFuture` (`command.date > LocalDate.now(clock.withZone(zoneId))`), `Positive` (`amount.signum() <= 0`) and `UnsupportedCurrency` (`currency != configured`).
  - `fingerprint(command): String`: SHA-256 hex of `date|merchant.trim()|amount.setScale(2).toPlainString()|currency|categoryId|accountId`.

  Make T031 green · skill: backend-developer
- [X] T040 [US3] Extend `…/main/repository/spending/SpendingRepository.kt` with `findByIdAndUserId` and `findByUserIdAndIdempotencyKey`. Create `…/main/repository/spending/SpendingMappings.kt`, where `toModel(categoryName, accountName, currency)` builds `Spending` · skill: backend-developer
- [X] T041 [US3] Create `…/main/service/SpendingService.kt` (`@Transactional(readOnly = true)` class):
  - **`create(identity, command, idempotencyKey: String?): CreateResult`**:
    1. Resolve the user.
    2. Collect the `SpendingRules` violations, plus `NotFound` at `#/categoryId` and `#/accountId` when `categoryRepository`/`accountRepository.findByIdAndUserId` returns nothing. Throw a single `ValidationException` if any.
    3. If a key is given: an existing row with the same fingerprint gives `replayed = true`; with a different fingerprint, throw `UnprocessableEntityException("The Idempotency-Key was already used for a different spending", mapOf("reason" to "idempotency-key-reused"))`.
    4. Save with `saveAndFlush`. On a `DataIntegrityViolationException` with a key, reload by key and repeat step 3.
  - **`get`**, **`update(id, command)`** (same validation; sets `updatedAt = Instant.now(clock)`) and **`delete(id)`**.
  - Every returned spending carries the current category and account names (FR-034).
  - Make T032 green · skill: backend-developer
- [X] T042 [US3] Create `…/main/controller/SpendingController.kt` at `/api/v1/spendings`, tag `Spendings`:
  - **`createSpending`**: `@RequestHeader("Idempotency-Key", required = false) @Size(min = 1, max = 255) @Pattern(regexp = "^[\\x21-\\x7E]+$")`, with the class annotated `@Validated`. Returns `ResponseEntity` 201 + `Location`, or 200 + `Location` + `Idempotent-Replayed: true` when replayed.
  - **`getSpending`**, **`updateSpending`** (`PUT`, `@Valid` body) and **`deleteSpending`** (`@ResponseStatus(NO_CONTENT)`).
  - Every `@ApiResponse` exactly as in `contracts/openapi.yaml`, including the 200 replay response and the `SpendingLoblaws` example.
  - Make T033, T034 and T035 green · skill: backend-developer

**Checkpoint**: US1–US3 work. A new user records their first spending in 2 requests
(SC-001), and amounts read back exactly (SC-005).

---

## Phase 6: User Story 4 — View my spendings (Priority: P1)

**Goal**: A paged, filtered, sorted spending list with a stable order and explicit rejection of
bad parameters.

**Independent Test**: 30 spendings across 2 months, 2 accounts and 3 categories. The first page
holds the newest. Paging through returns each spending exactly once. Each filter returns exactly
its matches, and bad parameters give 422.

### Tests for User Story 4 ⚠️

- [X] T043 [P] [US4] Write `…/test/dto/spending/SpendingSortTest.kt` (or put it in `SpendingServiceTest`): `"date,desc"`, `"date,asc"`, `"amount,desc"` and `"amount,asc"` parse. Null gives `date,desc`. `"merchant,asc"`, `"date"` and `"date,up"` are rejected with code `Unknown` at the `sort` parameter, with a message listing the allowed values · skill: backend-developer
- [X] T044 [P] [US4] Write `…/test/controller/SpendingListIT.kt` with 30 spendings for user A (September and October 2026, accounts Amex and Cash, categories Groceries, Taxi and Rent, including equal amounts and equal dates) and 5 for user B:
  - **Default**: `GET /api/v1/spendings` gives `size: 20`, `page: 0`, `totalItems: 30`, `totalPages: 2`, with items ordered by date desc, then `createdAt` desc, then `id`.
  - **Paging**: the union of `page=0,1` with `size=7` across all 5 pages equals all 30 ids with no repeat, for every sort value.
  - **Date range**: `from=2026-10-01&to=2026-10-31` returns only October and includes both end dates. `from` alone and `to` alone also work.
  - **Category and account filters**: `categoryId=<Groceries>&categoryId=<Taxi>` returns exactly those two. Adding `&accountId=<Amex>` gives the intersection (AND).
  - **Sort by amount**: `sort=amount,desc` orders by amount and stays stable across pages for equal amounts.
  - **Empty results**: an unknown `categoryId` gives 200 with `items: []` and `totalItems: 0`. A brand-new user also gives an empty page.
  - **Rejections**: `size=0`, `size=101`, `page=-1`, `from=2026-11-01&to=2026-10-01`, `sort=merchant,asc` and `from=2026-13-01` each give 422 with an `errors` entry naming that parameter.
  - **Isolation**: user B never sees user A's spendings · skill: backend-developer
- [X] T045 [P] [US4] Write `…/test/controller/SpendingListPerformanceIT.kt` (SC-004):
  - Batch-insert 10,000 spendings for one user via `jdbc` in under 30 s.
  - Assert that `GET /api/v1/spendings?from=2026-01-01&to=2026-06-30&categoryId=<x>&sort=amount,desc&size=100` returns 200 in under 1 s, after one warm-up call.
  - It runs inside `./gradlew check`, with no tag · skill: backend-developer

### Implementation for User Story 4

- [X] T046 [P] [US4] Create `…/main/model/SpendingFilter.kt` (`from: LocalDate?`, `to: LocalDate?`, `categoryIds: Set<UUID>`, `accountIds: Set<UUID>`), `…/main/model/SpendingPage.kt` (`items: List<Spending>, page, size, totalItems, totalPages`) and the `SpendingSort` enum (`DATE_DESC, DATE_ASC, AMOUNT_DESC, AMOUNT_ASC`, each with its API value). Create `…/main/dto/spending/SpendingSort.kt` with the parsing from T043 · skill: backend-developer
- [X] T047 [US4] Make `SpendingRepository` extend `JpaSpecificationExecutor<SpendingEntity>`. Create `…/main/repository/spending/SpendingSpecifications.kt`: `forUser(userId)` (always applied), `dateFrom`, `dateTo`, `categoryIn` and `accountIn`, each skipped when empty or null. Add a mapping from `SpendingSort` to `Sort` with the tie-break: `DATE_*` → `date` then `createdAt` then `id`, all in the same direction; `AMOUNT_*` → `amount` then `createdAt` then `id` (research R-010) · skill: backend-developer
- [X] T048 [US4] Add `list(identity, filter, sort, page, size): SpendingPage` to `…/main/service/SpendingService.kt`:
  - If `from > to`, throw `ValidationException` with a violation at `from`, code `Range`, message "must not be after to".
  - Query the page.
  - Resolve names from one `accountRepository` and one `categoryRepository` lookup of the caller's ids (3 queries per page, no N+1).
  - Map to `SpendingPage` · skill: backend-developer
- [X] T049 [P] [US4] Create `…/main/dto/spending/SpendingPageResponse.kt` (`items`, `page`, `size`, `totalItems`, `totalPages`) and its mapping · skill: backend-developer
- [X] T050 [US4] Add `GET ""` (`operationId = "listSpendings"`) to `…/main/controller/SpendingController.kt` with these query parameters:
  - `from` and `to` (`@DateTimeFormat(iso = DATE)`, optional)
  - `categoryId` and `accountId` (`List<UUID>?`, repeatable)
  - `page` (`@Min(0)`, default 0)
  - `size` (`@Min(1) @Max(100)`, default 20)
  - `sort` (`String?`, parsed by `SpendingSort`)

  Do **not** use Spring `Pageable` or `max-page-size` clamping. Document every parameter as in the contract, with responses 200 (example), 401, 422, 500 and 503. Make T043, T044 and T045 green · skill: backend-developer

**Checkpoint**: All P1 stories work. A user can record and view spendings end to end.

---

## Phase 7: User Story 5 — Tailor my categories (Priority: P2)

**Goal**: Create, rename and delete custom categories. Defaults are immutable, and in-use
categories cannot be deleted.

**Independent Test**: Create "Coffee", record a spending in it, rename it to "Cafés" and check
that the spending shows "Cafés". Deleting "Coffee" while it is in use gives 409. Renaming or
deleting "Groceries" gives 422. Deleting an unused custom category gives 204.

### Tests for User Story 5 ⚠️

- [X] T051 [P] [US5] Extend `…/test/service/CategoryServiceTest.kt` (MockK):
  - **Create**: makes `isDefault = false` and follows the Name rule (`duplicate-name`, including against a default's name such as `"groceries"`).
  - **Defaults**: rename or delete of a default gives `UnprocessableEntityException("<name> is a default category and cannot be changed", mapOf("reason" to "default-category"))`. This is checked **before** the in-use check.
  - **Custom**: rename of a custom category follows the Name rule. Delete when in use gives `ConflictException` with `in-use` and `spendingCount`; delete with 0 spendings removes the row.
  - **Not found**: a missing or foreign id gives `NotFoundException` · skill: backend-developer
- [X] T052 [P] [US5] Extend `…/test/controller/CategoryControllerIT.kt` (create it if T019 did not):
  - **Create**: `POST /api/v1/categories {"name":"Coffee"}` gives 201, `Location` and `isDefault: false`.
  - **Duplicates**: `{"name":"groceries"}` gives 409 `duplicate-name`.
  - **Validation**: invalid names give 422 at `#/name`.
  - **Rename**: `PUT /{coffee} {"name":"Cafés"}` gives 200, and a spending in that category then shows `category.name: "Cafés"` (FR-034).
  - **Defaults**: `PUT` and `DELETE` on Groceries give 422 `default-category`, and Groceries is unchanged.
  - **Delete**: `DELETE` of an unused custom category gives 204; a repeat gives 404.
  - **Other users**: user B on user A's custom category gives 404 for `GET`, `PUT` and `DELETE`.
  - **Auth**: no token gives 401 · skill: backend-developer
- [X] T053 [P] [US5] Extend `…/test/controller/DeleteInUseIT.kt` (categories part). With 1 spending in "Coffee", `DELETE` gives 409 `{reason: "in-use", spendingCount: 1}` and `GET /categories` shows `spendingCount: 1`. After `PUT`ting the spending to "Groceries", the delete gives 204 · skill: backend-developer

### Implementation for User Story 5

- [X] T054 [US5] Extend `…/main/repository/category/CategoryRepository.kt` with `findByIdAndUserId`, `existsByUserIdAndNameIgnoreCase` and `existsByUserIdAndNameIgnoreCaseAndIdNot`. Add `create(NameCommand)`, `rename(id, NameCommand)` and `delete(id)` to `…/main/service/CategoryService.kt`, following T028's pattern (Name rule, not-yours = not found with resource `category`, in-use via `spendingRepository.countByUserIdAndCategoryId` and `ConflictException`, FK race catch), with the default-category refusal first. Make T051 green · skill: backend-developer
- [X] T055 [US5] Add `POST ""` (`createCategory`: `ResponseEntity` 201 + `Location`; 400, 401, 409, 422, 500, 503), `PUT "/{categoryId}"` (`renameCategory`: 200, 400, 401, 404, 409, 422 `ValidationOrDefaultCategory`, 500, 503) and `DELETE "/{categoryId}"` (`deleteCategory`: 204, 400, 401, 404, 409 `InUse`, 422 `DefaultCategory`, 500, 503) to `…/main/controller/CategoryController.kt`, exactly as in the contract. Make T052 and T053 green · skill: backend-developer

**Checkpoint**: All five stories work on their own.

---

## Phase 8: Polish & Cross-Cutting Concerns

- [X] T056 [P] Write `…/test/controller/OwnershipIT.kt` (SC-003). User A creates an account, a custom category and a spending. Then, with user B's token:
  - Every path operation on A's ids (`GET`, `PUT`, `DELETE` on `/accounts/{id}`, `/categories/{id}` (custom and default) and `/spendings/{id}`) gives 404 `not-found`, with a body identical to an unknown random UUID's apart from ids and `traceId`.
  - `POST` and `PUT /spendings` referencing A's account or category give 422 `NotFound` at the pointer.
  - B's list endpoints contain none of A's ids.
  - Afterwards, A's rows are byte-for-byte unchanged (compared via `jdbc`) · skill: backend-developer
- [X] T057 Extend `…/test/config/OpenApiExportIT.kt`:
  - Assert the 15 new operationIds exist under the 6 paths.
  - Assert each one declares exactly the response codes listed in `specs/002-spending-tracking/contracts/openapi.yaml`.
  - Assert `Money.amount` is `type: string` with the pattern, `Idempotency-Key` is documented on `createSpending`, and every error response has an example.
  - Assert `PUT /api/v1/users/me` is unchanged from 001's responses (FR-041).
  - Run the test to regenerate `hl-budget-backend/contracts/api.yaml`, then compare it by hand with the design contract. Fix drift in the annotations, never in the generated file. Commit it (SC-006) · skill: backend-developer
- [X] T058 [P] Update `hl-budget-backend/README.md`: the new endpoints table, the two required environment variables (and that startup fails without them), the 422-vs-400 rule and the `Idempotency-Key` behaviour · skill: backend-developer
- [X] T059 Run `./gradlew check` in `hl-budget-backend/`: ktlint, detekt, all tests, and Kover 80% line / 75% branch overall plus 75% line per package. Fix anything red. Then run the local manual journey in `specs/002-spending-tracking/quickstart.md` §2–§3 (rows 1–18), including the startup failure with `HLBUDGET_MONEY_CURRENCY=XYZ` · skill: backend-developer
  - Done 2026-10-27: gate green (252 tests); startup failure confirmed (`hlbudget.money.currencyValid`). The curl journey needs a real Auth0 token and was not run by hand; every row is covered by the ITs, which run the real app with signed tokens. Repeat rows 1, 3 and 5 against the deployed host in T063.
- [X] T060 [P] Frontend types only: copy `hl-budget-backend/contracts/api.yaml` to `hl-budget-frontend/contracts/api.yaml`, run `pnpm generate:api` (updates `hl-budget-frontend/app/lib/api-schema.d.ts`), then `pnpm typecheck && pnpm lint && pnpm test`. No other frontend change. Commit on a `002-spending-tracking` branch in `hl-budget-frontend` (research R-001) · skill: frontend-developer
- [X] T061 Operator step (ask the user to confirm `CAD` and `America/Toronto` first): in `~/Documents/Projects/hl-helm/charts/apps/hl-budget/values.yaml`, add `- {name: HLBUDGET_MONEY_CURRENCY, value: "CAD"}` and `- {name: HLBUDGET_TIMEZONE, value: "America/Toronto"}` to the backend `env`, update `values.schema.json` if it lists env names, and open the PR. **Merge it before the backend release** (quickstart §5) · skill: homelab:hl-service-config
- [X] T062 With the user's agreement, open and merge the `hl-budget-backend` PR (`002-spending-tracking` → `main`), then the frontend PR. Merge the release workflow's image-bump PR in hl-helm · skill: backend-developer
- [X] T063 Verify the rollout: Argo CD `hl-budget` is Synced and Healthy, the backend pod is Ready, the log shows Flyway `V2` and `V3` applied, and there are no startup errors. Then repeat quickstart §3 rows 1, 3 and 5 against the deployed host with a real token (quickstart §5 step 4) · skill: homelab:hl-deploy-status
- [X] T064 [P] Mark `specs/002-spending-tracking/spec.md` **Status** as Implemented with the date, and note any deferred item · skill: none

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: T001 blocks T002. T003 can run alongside T002.
- **Foundational (Phase 2)**: depends on Setup and blocks every story.
  - T008 → T009, T010 run in any order.
  - T011 → T012 → T013 → T014.
  - T015, T016 and T017 can run in parallel after T013.
- **US1 (Phase 3)**: after Phase 2.
- **US2 (Phase 4)**: after Phase 2. It is independent of US1; it only needs the `categories` table to exist, not the seeding.
- **US3 (Phase 5)**: after US1 (categories to reference) and US2 (accounts to reference, `SpendingRepository` from T027).
- **US4 (Phase 6)**: after US3 (spendings to list, `SpendingService`/`SpendingController`).
- **US5 (Phase 7)**: after US1 (`CategoryService`/`CategoryController`). Its in-use and rename-reflection tests (T052, T053) need US3.
- **Polish (Phase 8)**: after all stories. T061 must merge before T062's release. T063 comes after T062.

### User Story Dependencies

```text
Phase 2 ──┬── US1 ──┬──────────── US5 (T052/T053 also need US3)
          └── US2 ──┴── US3 ── US4
```

### Within Each User Story

- Tests are written first and fail, then models, repositories, services, DTOs and the controller.
- Each controller task is done when its story's ITs are green.

### Parallel Opportunities

- **Phase 2 tests**: T004, T005, T006 and T007 touch different files.
- **Phase 2 implementation**: T015, T016 and T017 can run together after T013.
- **US1 and US2**: once Phase 2 is done, they can be built in parallel by different people or sessions.
- **Within each story**: all `[P]` test tasks together, then models and DTOs (`[P]`) alongside repository work.
- **Polish**: T056, T058, T060 and T064 can run in parallel. T060 needs T057's regenerated contract.

---

## Parallel Example: User Story 3

```text
# Tests first, all different files:
T031 SpendingRulesTest.kt
T032 SpendingServiceTest.kt
T033 SpendingControllerIT.kt
T034 IdempotencyIT.kt
T035 DeleteInUseIT.kt (accounts)

# Then the independent building blocks together:
T036 model/Spending.kt, SpendingCommand.kt
T037 dto/MoneyDto.kt, RefDto.kt
T038 dto/spending/SpendingRequest.kt, SpendingResponse.kt, SpendingMappings.kt

# Then, in order: T039 SpendingRules → T040 repository → T041 SpendingService → T042 SpendingController
```

## Parallel Example: User Stories 1 and 2

```text
# After Phase 2:
Session A: T018 → T019 → T020 → T021/T023 → T022 → T024   (categories, read-only + seeding)
Session B: T025/T026 → T027/T029 → T028 → T030            (accounts CRUD)
```

---

## Implementation Strategy

### MVP First (User Story 1)

1. Phase 1, then Phase 2: the foundation is green.
2. Phase 3 (US1): every user sees their 17 defaults. **Stop and validate** with T019.

US1 alone is a deployable increment, but it is not useful to a person on its own. The first
release worth deploying is **US1 + US2 + US3 + US4** (all P1): record and view spendings.

### Incremental Delivery

1. Foundation → US1 → validate (defaults).
2. US2 → validate (accounts).
3. US3 → validate (record a spending; SC-001, SC-005, SC-007).
4. US4 → validate (view spendings; SC-004). **Release candidate**: Polish T056–T063.
5. US5 → validate (custom categories). This can ship in the same release or right after it.

---

## Notes

- `[P]` means different files with no dependency on an unfinished task.
- Before changing behaviour, a test for it fails first.
- Never edit `V1__create_users.sql`, or V2/V3 once they have been applied anywhere. A fix is a new
  `V4`.
- No entity outside `repository/` and `service/`; no DTO inside `service/`; `ResponseEntity` only
  for `Location` and `Idempotent-Replayed`.
- The 400→422 change (T010) is the only change to existing behaviour. Every other change is
  additive (FR-041).
