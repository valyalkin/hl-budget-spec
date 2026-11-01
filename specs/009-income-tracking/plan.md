# Implementation Plan: Income

**Branch**: `009-income-tracking` | **Date**: 2026-11-01 | **Spec**: [spec.md](./spec.md)

**Input**: Feature specification from `/specs/009-income-tracking/spec.md`

## Summary

An Income tab next to Spendings, on desktop and phone, with the same screens and rules, its own
categories (Salary and Investments by default) and the shared accounts.

**Backend** (`hl-budget-backend`, v1, research R-001 – R-004):
- Flyway V7: tables `income_categories`, `incomes`, `income_imports`, and
  `users.income_categories_seeded_at`. Incomes reference income categories and the existing
  accounts through composite keys, so an income can never use a spending category.
- Salary `#1e5b45` and Investments `#1d4ed8` are seeded once per user on the first call; custom
  income categories get `CategoryPalette.next` over income categories.
- New resources: `/incomes`, `/income-categories`, `/income-sources`, `/monthly-income-totals`,
  `/income-category-totals`, `/income-account-totals`, `/income-imports`. `Account` gains
  `incomeCount`; the account in-use refusal also counts income.
- The SQL for totals and past names, and the date and amount rules, are shared with spendings
  through a `Ledger` enum; entities, DTOs and controllers are thin twins.

**Frontend** (`hl-budget-frontend`, R-005 – R-009):
- A new `/income` route beside `/spendings`, with "Spendings | Income" tabs (header nav on the
  desktop, a segmented control on the phone) that keep `?month=`.
- The existing Spendings components take a `ledger` prop (default `'spendings'`) driven by one
  `LEDGER` table of labels, messages and storage keys; data and actions get income twins.
- Settings: Spending categories (renamed), Income categories (new), Accounts (both counts).
- CSV import on the Income tab with the same reader and review.

## Technical Context

**Language/Version**: Kotlin 2 / Java 21 (backend); TypeScript 5 strict (frontend).

**Primary Dependencies**: Spring Boot 4, JPA/Hibernate, JdbcClient, Flyway, springdoc (backend);
Next.js App Router, React, Tailwind v4, zod, `openapi-typescript` (frontend). No new dependencies.

**Storage**: PostgreSQL. Three new tables and one new column via V7; indexes on `incomes` mirror
`spendings`. Existing tables are unchanged apart from the new `users` column.

**Testing**: JUnit 5 + Testcontainers Postgres, Kover (backend); Vitest + Testing Library,
Playwright with desktop and phone projects and axe (frontend).

**Target Platform**: homelab Kubernetes (Argo CD); browsers on desktop and phone.

**Project Type**: web application: two services in two repositories, with this spec repo
alongside.

**Performance Goals**: tab switch or month change on Income within 1 s for 10,000 spendings and
1,000 incomes (SC-004); updates within 1 s after a change (SC-005). The Income page makes the same
number of API calls as the Spendings page, in the same two parallel rounds.

**Constraints**: Spendings must not change (FR-004): its routes, components' default behaviour,
storage keys, wording and e2e specs stay as they are. Additive API changes only. Phone tab switch
fits 320px wide with 44px targets.

**Scale/Scope**: one user (a handful at most); income entries a few per month, a few income
categories.

All unknowns are resolved in [research.md](./research.md).

## Skills that drive the implementation

| Skill | Owns | Used in |
|---|---|---|
| `backend-developer` | V7 migration and IT, income entities, repositories, services, controllers with OpenAPI docs, `Ledger` refactor of shared queries and rules, account `incomeCount`, tests, `contracts/api.yaml` export, `./gradlew check` | Phase A |
| `frontend-developer` | Type regeneration, `LEDGER`, `/income` route, tabs, ledger-aware components, data and actions, Settings, import, unit tests, e2e, pnpm gates | Phase B |
| `homelab:hl-deploy-status` | Verify both rollouts | Deploy |

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-checked after Phase 1 design.*

| Principle | Check | Result |
|---|---|---|
| I. Spec-driven | Spec 009 → plan → tasks before code | Pass |
| II. Separate services, contract-first | Seeding, matching, sums and colours live in the backend; the frontend renders and maps one field name (R-006, R-007). Types are regenerated from the exported OpenAPI | Pass |
| III. Secure by default | Every new endpoint is behind the existing JWT filter; every query filters on the caller's `user_id`; composite FKs keep an income on the caller's own category and account; `OwnershipIT` covers each new endpoint (SC-006). `/income` is `noindex` like `/spendings` | Pass |
| IV. Tests and gates | Migration, seeding, controller, import, totals, ownership and performance ITs; unit tests for `LEDGER`, data mapping, actions and income rendering; e2e for every income flow on desktop and phone with axe; existing specs unchanged and green. Both gates must pass | Pass |
| V. Homelab-ready | New migration V7 only; V1–V6 untouched. Deploy through release → hl-helm → Argo CD. No new config or secrets | Pass |
| VI. Simplicity | No new service, dependency or infrastructure. Shared SQL and rules, thin twins elsewhere (R-004); one set of UI components with a `ledger` prop (R-006) | Pass |
| VII. Versioned API | New plural kebab-case resources under `/api/v1`, correct methods and statuses, `Idempotency-Key` on both creates, all documented with examples; `Account.incomeCount` and the extra `details.incomeCount` are additive; nothing removed or renamed. Three totals endpoints and the income category list are not paginated (justified below, as in 003 and 008) | Pass with justification |

**Post-design re-check**: unchanged. The contract delta ([contracts/openapi-delta.yaml](./contracts/openapi-delta.yaml))
is additive only. Backward compatibility: **compatible**; the only behaviour change on an existing
operation is that `DELETE /api/v1/accounts/{id}` now also refuses (with the existing `409 in-use`)
when income uses the account, which is the same refusal reason the client already handles.

## Project Structure

### Documentation (this feature)

```text
specs/009-income-tracking/
├── plan.md
├── research.md
├── data-model.md
├── quickstart.md
├── contracts/openapi-delta.yaml
├── checklists/requirements.md
└── tasks.md              # /speckit-tasks
```

### Source Code

```text
hl-budget-backend/
├── src/main/resources/db/migration/
│   └── V7__create_incomes.sql                          # new (R-001, data-model.md)
├── src/main/kotlin/com/valyalkin/hlbudget/
│   ├── model/
│   │   ├── Ledger.kt                                   # new: table/column names per kind (R-004)
│   │   ├── DefaultIncomeCategories.kt                  # new: Salary, Investments + colours (R-002)
│   │   ├── Income.kt, IncomeCommand.kt, IncomeCategory.kt, IncomeImport.kt   # new
│   │   ├── Account.kt                                  # + incomeCount
│   │   └── Totals.kt                                   # + income totals and IncomeSource
│   ├── repository/
│   │   ├── CurrentUserRepository.kt                    # + claimIncomeSeeding, insertDefaultIncomeCategories
│   │   ├── account/AccountRepository.kt                # AccountRow + incomeCount
│   │   ├── income/                                     # new: IncomeEntity, IncomeRepository, IncomeSpecifications,
│   │   │                                               #      IncomeInserts, mappings
│   │   ├── incomecategory/                             # new: IncomeCategoryEntity, repository, mappings
│   │   ├── incomeimport/                               # new: IncomeImportEntity, repository, inserts
│   │   └── spending/TotalsQueries.kt, MerchantQueries.kt   # take a Ledger
│   ├── service/
│   │   ├── CurrentUserService.kt                       # seeds income defaults once
│   │   ├── SpendingRules.kt → EntryRules.kt            # shared date/amount/currency rules
│   │   ├── NamedItemResolver.kt                        # category lookup per ledger; income messages
│   │   ├── NamedItemErrors.kt                          # inUse with spending and income counts
│   │   ├── AccountService.kt                           # in-use counts income too
│   │   ├── IncomeService.kt, IncomeCategoryService.kt, IncomeImportService.kt, IncomeSourceService.kt  # new
│   │   └── TotalsService.kt                            # + income monthly, by category, by account
│   ├── controller/
│   │   ├── IncomeController.kt, IncomeCategoryController.kt, IncomeImportController.kt,
│   │   │   IncomeSourceController.kt                   # new
│   │   └── TotalsController.kt                         # + three income endpoints
│   └── dto/income/, dto/incomecategory/, dto/incomeimport/, dto/totals/, dto/account/   # new + incomeCount
└── src/test/kotlin/com/valyalkin/hlbudget/
    ├── repository/IncomeMigrationIT.kt                 # V6 → V7: tables, constraints, data untouched
    ├── service/                                        # IncomeService, IncomeCategoryService, seeding, EntryRules tests
    └── controller/                                     # IncomeControllerIT, IncomeCategoryControllerIT, IncomeImportIT,
                                                        # IncomeTotalsIT, IncomeSourceIT, IncomePerformanceIT;
                                                        # AccountControllerIT, OwnershipIT, OpenApi export updated

hl-budget-frontend/
├── app/income/page.tsx, loading.tsx, error.tsx         # new route (R-005)
├── app/lib/
│   ├── api-schema.d.ts                                 # regenerated
│   ├── definitions.ts                                  # income aliases
│   ├── ledger.ts                                       # new: Ledger type and LEDGER table (R-006)
│   ├── data.ts                                         # loadLedgerPage, income readers, getEarliestYear (R-007)
│   ├── actions.ts                                      # saveIncome, deleteIncome, importIncome, income categories
│   ├── messages.ts                                     # income wording; account usage with both counts (R-008)
│   └── csv-import.ts                                   # source column pattern (R-009)
├── app/ui/app-header.tsx                               # Income link, keeps ?month=
├── app/ui/ledger-tabs.tsx                              # new: tab links (desktop nav, phone segmented)
├── app/ui/spendings/*                                  # month-section, month-line, spendings-table, entry-row,
│                                                       # spending-row, month-total-card, breakdown-* take `ledger`
├── app/ui/import/*                                     # provider and dialogs take `ledger`; income sample file
├── app/ui/settings/                                    # settings-dialog sections; income-categories-panel.tsx (new);
│                                                       # accounts-panel both counts
├── app/ui/phone/*                                      # phone-header tabs; section, list, form, delete sheet,
│                                                       # breakdown, review, settings take `ledger`
├── public/sample-income.csv                            # new, beside sample-spendings.csv
└── e2e/income/*.spec.ts, e2e/phone/income/*.spec.ts    # new; a11y specs cover /income
```

**Structure Decision**: same two service repositories; no new modules. Income lives in new files on
the backend; on the frontend the Spendings components are reused with a `ledger` prop, keeping
their file names.

## Delivery order

1. **Branches**: `009-income-tracking` from `origin/main` in both repos (this spec branch exists).
2. **Phase A, backend** (blocks the frontend's types):
   - A1: V7 migration and `IncomeMigrationIT`; `Ledger` refactor of `TotalsQueries`,
     `MerchantQueries`, `SpendingRules` → `EntryRules`, with the existing spending ITs green and
     unchanged.
   - A2: income categories: entity, repository, seeding in `CurrentUserService`, service,
     controller; seeding and controller ITs.
   - A3: incomes: entity, repository, specifications, `IncomeService` with idempotent create,
     resolver for income categories, controller; `IncomeControllerIT`, `OwnershipIT` cases.
   - A4: accounts: `incomeCount` and the in-use refusal counting income; `AccountControllerIT`.
   - A5: totals and sources endpoints; `IncomeTotalsIT`, `IncomeSourceIT`, `IncomePerformanceIT`.
   - A6: income imports; `IncomeImportIT`.
   - A7: export `contracts/api.yaml`, `./gradlew check`, PR, release, `hl-deploy-status`.
3. **Phase B, foundation**: regenerate types; `LEDGER`; `loadLedgerPage` with `loadSpendingsPage`
   as a wrapper; `ledger` prop through the shared components with `'spendings'` default; all
   existing unit tests and e2e stay green.
4. **US1 view**: `/income` route, tabs on desktop, month line, table, footer, total card; e2e.
5. **US2 add/edit/delete on desktop**: income actions, entry row with Source and income
   categories, autofill from sources; e2e incl. keyboard entry.
6. **US3 breakdowns**: By category and By account on Income with their own storage keys; e2e.
7. **US4 Settings**: Spending categories rename, Income categories panel, account counts and
   refusal wording; e2e desktop and phone.
8. **US6 phone**: header tabs, income list, Add / Edit income, delete sheet, breakdowns, back
   gesture; phone e2e at 390 and 360 wide.
9. **US5 import**: ledger-aware import, income sample, Source column; e2e desktop and phone.
10. **Gates and deploy**: pnpm gates and `pnpm e2e`, frontend PR, release, `hl-deploy-status`, live
    check (quickstart §5), README update.

## Complexity Tracking

| Violation | Why Needed | Simpler Alternative Rejected Because |
|---|---|---|
| `GET /api/v1/income-category-totals`, `/income-account-totals` and `/income-categories` are not paginated (Principle VII: "Collection endpoints MUST be paginated") | Each returns at most the user's income category or account count (a handful), and the screen needs the whole set at once for shares, totals and dropdowns; same justification as `/category-totals`, `/account-totals` and `/categories` in the 002, 003 and 008 plans | Paging would make the client loop for no gain. Each response uses a named `items` field, so paging parameters can be added later as optional |
| `GET /api/v1/monthly-income-totals` returns a fixed twelve-item list without paging | Always exactly twelve months, like `/monthly-totals` | Paging a fixed-size list adds nothing |
