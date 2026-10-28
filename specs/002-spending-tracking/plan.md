# Implementation Plan: Spending Tracking (Backend)

**Branch**: `002-spending-tracking` | **Date**: 2026-10-27 | **Spec**: [spec.md](./spec.md)

**Input**: Feature specification from `/specs/002-spending-tracking/spec.md`

## Summary

This feature adds HL Budget's first business data to `hl-budget-backend`: **accounts**,
**categories** and **spendings**, each with list, fetch, create, update (`PUT`) and delete under
`/api/v1`, all scoped to the caller's Auth0 `sub`.

- **Identity and defaults**: every budgeting call resolves the caller to `users.id`, creating
  the row if it is missing. On the caller's first call it seeds the 17 default categories
  exactly once, guarded by a `users.categories_seeded_at` flag set under a row lock (R-002,
  R-003).
- **Integrity in the database**:
  - Composite foreign keys keep a spending pointing only at its own user's account and category.
  - `NO ACTION` foreign keys plus a service count refuse deleting an item that spendings still
    use (409 `in-use`, R-005).
  - Case-insensitive unique indexes enforce unique names (409 `duplicate-name`, R-004).
  - Default categories are immutable (422 `default-category`).
- **Money**: amounts are exact `numeric(10,2)`, exchanged as `{ amount: "84.37", currency }`
  in the single configured currency. Dates after "today" in the configured time zone are
  rejected (R-006).
- **Idempotent create**: `POST /spendings` takes an optional `Idempotency-Key`, stored on the
  spending row (R-007).
- **Listing and counts**: the spending list is offset-paged (20 by default, at most 100),
  filterable by date range, category and account, and sortable by date or amount with a stable
  tie-break (R-010). Accounts and categories carry a live `spendingCount` (R-011).
- **Status codes**: field-level validation moves from 400 to 422 to match Principle VII (R-009).
- **Elsewhere**: the frontend only regenerates its API types (R-001), and hl-helm gains two
  environment variables.

## Technical Context

**Language/Version**: Kotlin 2.3 on Java 21 (unchanged from 001)

**Primary Dependencies**: Spring Boot 4.1 (webmvc, data-jpa, validation, security OAuth2
resource server, flyway), springdoc-openapi 3.1. **No new dependencies**: Spring Data
`Specification` is part of data-jpa, and SHA-256 comes from the JDK.

**Storage**: PostgreSQL 17. The migrations add `V2` (`users.categories_seeded_at`) and `V3`
(`accounts`, `categories`, `spendings`). Flyway owns the schema with `ddl-auto: validate`.

**Testing**: JUnit 5, MockK, AssertJ, Testcontainers Postgres 17, `RestTestClient`, and WireMock
for JWKS (001's `JwtTestTokens`). Kover gates are 80/75 overall and 75 per package. The gate is
`./gradlew check`. The frontend gate runs after type regeneration.

**Target Platform**: The homelab k3s cluster, via the existing `charts/apps/hl-budget` (1 backend
replica)

**Project Type**: Web service (the backend API of a two-service web app)

**Performance Goals**: A filtered spending page for a user with 10,000 spendings returns in
< 1 s (SC-004). The design targets < 50 ms on Postgres.

**Constraints**:
- No floating-point money.
- Only additive contract changes in v1 (FR-041).
- Identity comes only from the token.
- New config is typed, required and validated at startup.
- Migrations are additive.

**Scale/Scope**: A handful of users; tens of accounts and categories each; up to about 10k
spendings per user. 15 operations on 6 paths, 3 tables, 1 column.

All unknowns are resolved in [research.md](./research.md) (R-001 … R-013). No
`NEEDS CLARIFICATION` remains.

## Skills that drive the implementation

| Skill | Owns | Used in |
|---|---|---|
| `backend-developer` | Everything in `hl-budget-backend`: migrations, entities and repositories, models, services, DTOs, controllers with full `@ApiResponse` sets, the error-package changes (R-009, `ValidationException`, `ConflictException`), typed config, tests, `contracts/api.yaml` export, `./gradlew check` | All stories |
| `frontend-developer` | Copy `contracts/api.yaml`, `pnpm generate:api`, pnpm gates. No UI | Final step |
| `homelab:hl-service-config` | hl-helm PR adding `HLBUDGET_MONEY_CURRENCY`, `HLBUDGET_TIMEZONE` to the backend env | Before the backend deploy |
| `homelab:hl-deploy-status` | Verify the rollout (Argo CD Healthy, Flyway V2/V3 applied) | After deploy |

Where a skill and the constitution disagree, the constitution wins. This plan makes two
reconciliations. **R-009**: field validation is 422, not the skill's 400. **R-006**: money is
a decimal string, not the skill example's `amountCents`.

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

| Principle | Requirement | Plan | Status |
|---|---|---|---|
| I. Spec-driven, foundation-first | Spec → plan → tasks before code; later features are independently deployable increments on the foundation | Spec clarified (6 answers); builds on 001's user record and auth unchanged; deployable on its own (API only) | PASS |
| II. Separate services, contract-first | Backend owns data, rules, contract; frontend types generated; RFC 9457 errors with stable `type` + `traceId`; 5xx leaks nothing | All rules in backend `service/`; [contracts/openapi.yaml](./contracts/openapi.yaml) → generated `contracts/api.yaml` → `pnpm generate:api` (R-001); every refusal is a `BusinessException` subclass through the existing handler | PASS |
| III. Secure by default | JWT on every request; all data scoped to `sub`; another user's resource answers 404; no secrets added | `CurrentUserService` resolves `userId` from the token only (R-002); every query filters on `user_id`; composite FKs prevent cross-user references in the DB; same 404/`NotFound` for "missing" and "not yours" (R-012); currency and time zone are non-secret config | PASS |
| IV. Tests and quality gates | Tests ship with code; Testcontainers, no H2; 80/75 + 75 per package; gate green locally and in CI | R-013 test map: unit per service, IT per resource plus ownership, concurrency-seeding, idempotency, in-use, paging and 10k-row performance ITs | PASS |
| V. Homelab-ready operations | Env-only config with `.env.example`; Flyway additive and immutable; deploy via release → hl-helm → Argo CD | Two new required env vars documented in `.env.example` and set in the chart first (quickstart §5); `V2`, `V3` additive; `V1` untouched | PASS |
| VI. Simplicity at personal scale | No queues, caches or extra services; additions justified | No new dependency, table for idempotency, counter cache or job: keys live on the spending row (R-007), counts are computed (R-011), defaults are a constant (R-003) | PASS |
| VII. Versioned, consistent API | `/api/v1`; plural kebab-case resources ≤ 1 level; method semantics; precise status codes; camelCase, `YYYY-MM-DD`, UTC timestamps, UUID ids; Money object with decimal string; paged collections with named items field and capped size; boundary validation with JSON Pointers; `Idempotency-Key` on money-moving creates; everything documented with examples | `/api/v1/{accounts,categories,spendings}[/{id}]`; `POST` 201 + `Location`, `PUT` 200, `DELETE` 204, 404/409/422 per R-004/R-005/R-009/R-012; `Money` per R-006; `SpendingPage.items` with `size` ≤ 100 and explicit rejection (R-010); `Idempotency-Key` (R-007); contract with examples for every response including errors. Account/category lists are not paged (see Complexity Tracking) | PASS (one justified deviation) |
| Tech & platform constraints | Listed stack; no float money (`NUMERIC`/`BigDecimal` with explicit currency) | Stack unchanged; `numeric(10,2)` + `BigDecimal` + configured ISO 4217 currency, which every response states | PASS |

**Contract compatibility statement** (Development Workflow): **backward compatible**, v1. The
change adds 6 paths and 15 operations. `PUT /api/v1/users/me` and its documented responses are
unchanged. The 400→422 move for field validation (R-009) affects no operation that 001 published,
because none takes a body or documents a validation response.

**Post-design re-check (after Phase 1)**: PASS. [data-model.md](./data-model.md) and
[contracts/openapi.yaml](./contracts/openapi.yaml) add nothing beyond the stack. The one
deviation, unpaged account and category lists, is recorded below. Operator step: confirm `CAD`
and `America/Toronto` before merging the hl-helm PR, because the currency cannot be changed later
without converting data (spec Assumptions).

## Project Structure

### Documentation (this feature)

```text
specs/002-spending-tracking/
├── plan.md              # This file
├── research.md          # Phase 0: decisions R-001 … R-013
├── data-model.md        # Phase 1: tables, constraints, validation rules, lifecycle
├── quickstart.md        # Phase 1: gates, local journey, frontend types, deploy order
├── contracts/
│   └── openapi.yaml     # The 15 new operations, schemas, error responses with examples
├── checklists/requirements.md
└── tasks.md             # Phase 2 (/speckit-tasks; not created here)
```

### Source Code (`hl-budget-backend`, worktree `hl-budget-spec/hl-budget-backend/`)

New files are marked `+`, changed files `~`. Layers above ~5 files use feature subpackages, as
the skill allows.

```text
hl-budget-backend/
├── .env.example                          ~ HLBUDGET_MONEY_CURRENCY, HLBUDGET_TIMEZONE
├── README.md                             ~ new endpoints and config
├── contracts/api.yaml                    ~ regenerated by OpenApiExportIT
└── src/
    ├── main/kotlin/com/valyalkin/hlbudget/
    │   ├── config/
    │   │   ├── HlBudgetProperties.kt     ~ money.currency (Currency), timeZone (ZoneId), validated
    │   │   ├── ClockConfig.kt            + Clock bean (UTC)
    │   │   └── OpenApiConfig.kt          ~ tags Accounts, Categories, Spendings; shared error responses
    │   ├── controller/
    │   │   ├── AccountController.kt      + /api/v1/accounts[/{accountId}]
    │   │   ├── CategoryController.kt     + /api/v1/categories[/{categoryId}]
    │   │   └── SpendingController.kt     + /api/v1/spendings[/{spendingId}], Idempotency-Key, list params
    │   ├── dto/
    │   │   ├── NameRequest.kt, MoneyDto.kt, RefDto.kt          +
    │   │   ├── account/   AccountResponse, AccountListResponse, AccountMappings     +
    │   │   ├── category/  CategoryResponse, CategoryListResponse, CategoryMappings  +
    │   │   └── spending/  SpendingRequest, SpendingResponse, SpendingPageResponse,
    │   │                  SpendingSort, SpendingMappings                            +
    │   ├── model/
    │   │   ├── Account.kt, Category.kt, DefaultCategories.kt                        +
    │   │   ├── Spending.kt (+ Money, Ref), SpendingCommand.kt, SpendingFilter.kt,
    │   │   │   SpendingPage.kt, NameCommand.kt                                      +
    │   ├── repository/
    │   │   ├── UserEntity.kt             ~ categoriesSeededAt
    │   │   ├── CurrentUserRepository.kt  + find-or-insert by subject; claim-seeding UPDATE … RETURNING
    │   │   ├── account/   AccountEntity, AccountRepository (+ count projection), AccountMappings    +
    │   │   ├── category/  CategoryEntity, CategoryRepository (+ count projection), CategoryMappings +
    │   │   └── spending/  SpendingEntity, SpendingRepository (JpaSpecificationExecutor),
    │   │                  SpendingSpecifications, SpendingMappings                                 +
    │   ├── service/
    │   │   ├── CurrentUserService.kt     + resolve(identity) → userId; seeds defaults once
    │   │   ├── AccountService.kt, CategoryService.kt                                +
    │   │   ├── SpendingService.kt        + create (idempotent), get, update, delete, list
    │   │   └── SpendingRules.kt          + today check, currency check, fingerprint
    │   └── error/
    │       ├── BusinessException.kt      ~ + ConflictException (409), ValidationException (422 + errors)
    │       └── GlobalExceptionHandler.kt ~ field validation → 422 (R-009); ValidationException rendering
    ├── main/resources/
    │   ├── application.yaml              ~ hlbudget.money.currency, hlbudget.time-zone (required, no default)
    │   ├── application-local.yaml        ~ CAD, America/Toronto
    │   └── db/migration/
    │       ├── V2__add_users_categories_seeded_at.sql               +
    │       └── V3__create_accounts_categories_spendings.sql         +
    └── test/
        ├── resources/application-test.yaml   ~ CAD, America/Toronto
        └── kotlin/com/valyalkin/hlbudget/
            ├── support/          ~ helpers: second user token, fixed Clock, seed builders
            ├── service/          + CurrentUserServiceTest, AccountServiceTest, CategoryServiceTest,
            │                       SpendingServiceTest, SpendingRulesTest
            ├── controller/       + AccountControllerIT, CategoryControllerIT, SpendingControllerIT,
            │                       SpendingListIT, DefaultCategoriesIT, OwnershipIT, IdempotencyIT,
            │                       DeleteInUseIT, SpendingListPerformanceIT
            ├── config/           ~ HlBudgetPropertiesTest (currency, time zone), OpenApiExportIT
            ├── error/            ~ GlobalExceptionHandlerIT/Test (400 vs 422 split)
            └── repository/       ~ SchemaValidationIT
```

Outside the backend:

```text
hl-budget-frontend/   contracts/api.yaml ~, app/lib/api-schema.d.ts ~ (generated). Nothing else
hl-helm/              charts/apps/hl-budget/values.yaml ~ two backend env entries
```

**Structure Decision**: The existing one-service-per-repo layout from 001, extended inside
`hl-budget-backend` using the package roles of the `backend-developer` skill. Feature
subpackages (`account/`, `category/`, `spending/`) are used only in `dto/` and `repository/`,
the layers that pass five files. No top-level feature packages, no `util/` or `mapper/`.

## Delivery order

1. **Branch**: `hl-budget-backend` worktree onto a new `002-spending-tracking` branch from
   `origin/main`. Resolve the open `fix/api-docs-off-in-cluster` branch first (R-001).
2. **Foundation** (blocks all stories):
   - Config: currency, time zone, `Clock`.
   - Error package: 422 split, `ValidationException`, `ConflictException`.
   - Migrations `V2`/`V3` with the entities.
   - `CurrentUserService` with seeding.
3. **US1** default categories: list categories plus `DefaultCategoriesIT`.
4. **US2** accounts: CRUD, counts, `in-use`.
5. **US3** record a spending: CRUD, money, date, references, idempotency.
6. **US4** view spendings: the list with filters, sort and paging, plus the performance IT.
7. **US5** custom categories: create, rename, delete with the default and in-use refusals.
8. **Contract**: regenerate `contracts/api.yaml`, compare it with
   [contracts/openapi.yaml](./contracts/openapi.yaml), then `./gradlew check`.
9. **Frontend types**: copy the contract and regenerate, then run the pnpm gates (quickstart §4).
10. **Deploy**: hl-helm env PR first, then the backend release, then `hl-deploy-status`
    (quickstart §5).

US1 and US2 can proceed in parallel once the foundation is in. US3 needs both. US4 and US5 need
US3.

## Complexity Tracking

| Violation | Why Needed | Simpler Alternative Rejected Because |
|-----------|------------|-------------------------------------|
| `GET /accounts` and `GET /categories` are not paginated (Principle VII: "Collection endpoints MUST be paginated") | FR-016 and FR-024 require all items in one response. A personal user has tens of them, and pickers need the full list | Paging them makes every client loop over pages for a list that fits in one screen. The response still uses a named `items` field, so `page`/`size` can be added later as optional parameters without a breaking change |
