# Implementation Plan: Spending by account

**Branch**: `008-account-breakdown` | **Date**: 2026-11-01 | **Spec**: [spec.md](./spec.md)

**Input**: Feature specification from `/specs/008-account-breakdown/spec.md`

## Summary

A By account breakdown next to By category, on desktop and phone, built from the same parts.

**Backend** (`hl-budget-backend`, v1, research R-001 – R-005):
- Flyway V6: `accounts.color` (`#rrggbb`, NOT NULL). Existing accounts are back-filled by creation
  order with the sequence `CategoryPalette.next` would have produced.
- A new account gets `CategoryPalette.next(colour counts of the caller's accounts)`.
- New required response field `Account.color` (additive).
- New endpoint `GET /api/v1/account-totals?from=&to=`, the twin of `/category-totals`, with
  schemas `AccountTotals` and `AccountTotal` (incl. `color`).

**Frontend** (`hl-budget-frontend`, R-006 – R-009):
- The desktop By category card, table and pie become one `BreakdownCard` / `BreakdownTable` /
  `BreakdownPie` set with a `kind` (`category` | `account`). A second card, By account, sits below
  By category in the summary column, with its own stored Table / Pie choice.
- On the phone, the switch gains "By account"; `CategoryBreakdown` becomes `Breakdown` with the
  same `kind`; a separate `accountChart` choice; `topCategories` → `topSlices`.
- `loadSpendingsPage` fetches account totals in parallel with category totals.

## Technical Context

**Language/Version**: Kotlin 2 / Java 21 (backend); TypeScript 5 strict (frontend).

**Primary Dependencies**: Spring Boot 4, JPA/Hibernate, JdbcClient, Flyway, springdoc (backend);
Next.js App Router, React, Tailwind v4, `openapi-typescript` (frontend). No new dependencies.

**Storage**: PostgreSQL. One new column (`accounts.color`) via V6. No new index: the totals query
filters on `ix_spendings_user_date`, as category totals do.

**Testing**: JUnit 5 + Testcontainers Postgres, Kover (backend); Vitest + Testing Library,
Playwright with desktop and phone projects and axe (frontend).

**Target Platform**: homelab Kubernetes (Argo CD); browsers on desktop and phone.

**Project Type**: web application: two services in two repositories, with this spec repo
alongside.

**Performance Goals**: month switch with the account breakdown within 1 s for 10,000 spendings
(SC-002); updates after a change within 1 s (SC-003). One extra parallel API call per page load.

**Constraints**: By category must not change (FR-001), including its stored Table / Pie choice
key. Additive API changes only. Account colours at least 3:1 against the card (palette already
checked in 007). Phone switch must fit 360px without sideways scroll.

**Scale/Scope**: one user (a handful at most), a few accounts each.

All unknowns are resolved in [research.md](./research.md).

## Skills that drive the implementation

| Skill | Owns | Used in |
|---|---|---|
| `backend-developer` | V6 migration and IT, colour on create, `Account.color`, `byAccount` query, `/account-totals` controller + OpenAPI docs, tests, `contracts/api.yaml` export, `./gradlew check` | Phase A |
| `frontend-developer` | Type regeneration, data loading, `Breakdown*` components, phone view and chart state, unit tests, e2e, pnpm gates | Phase B |
| `homelab:hl-deploy-status` | Verify both rollouts | Deploy |

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-checked after Phase 1 design.*

| Principle | Check | Result |
|---|---|---|
| I. Spec-driven | Spec 008 → plan → tasks before code | Pass |
| II. Separate services, contract-first | Sums and colour assignment live in the backend (R-001, R-004). The frontend only renders; types are regenerated from the exported OpenAPI | Pass |
| III. Secure by default | `/account-totals` is behind the existing JWT filter; the query and colour counts are scoped to the caller's `user_id`; an isolation IT checks a second user's spendings never appear | Pass |
| IV. Tests and gates | Migration IT, AccountService colour unit tests, `TotalsIT` and `TotalsPerformanceIT` for the new endpoint, OpenAPI export IT; Breakdown component and phone state unit tests; e2e for desktop and phone By account, plus axe. Both gates must pass | Pass |
| V. Homelab-ready | New migration V6 only; V1–V5 untouched. Deploy goes through release → hl-helm → Argo CD | Pass |
| VI. Simplicity | No new dependency, table or service. The desktop and phone breakdowns are generalised rather than copied | Pass |
| VII. Versioned API | One new versioned endpoint (plural kebab-case resource, `GET`, documented with examples) and one new response field; nothing removed or changed. Not paginated, as `/category-totals` (justified below) | Pass with justification |

**Post-design re-check**: unchanged. The contract delta ([contracts/openapi-delta.yaml](./contracts/openapi-delta.yaml))
is additive only.

## Project Structure

### Documentation (this feature)

```text
specs/008-account-breakdown/
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
│   └── V6__add_account_colors.sql                   # new (R-003)
├── src/main/kotlin/com/valyalkin/hlbudget/
│   ├── model/Account.kt                             # + color
│   ├── model/CategoryPalette.kt                     # KDoc: also used for accounts
│   ├── model/Totals.kt                              # + AccountTotal, AccountTotals
│   ├── repository/account/AccountEntity.kt          # + color (not updatable)
│   ├── repository/account/AccountRepository.kt      # AccountRow + color; colorCounts(userId)
│   ├── repository/account/AccountMappings.kt        # color through
│   ├── repository/spending/TotalsQueries.kt         # + AccountTotalRow, byAccount
│   ├── service/AccountService.kt                    # create assigns colour (R-002)
│   ├── service/TotalsService.kt                     # + byAccount (shares checkRange)
│   ├── controller/TotalsController.kt               # + listAccountTotals, example
│   ├── dto/account/AccountResponse.kt               # + color
│   ├── dto/account/AccountMappings.kt               # color through
│   └── dto/totals/TotalsResponses.kt                # + AccountTotalResponse, AccountTotalsResponse
└── src/test/kotlin/com/valyalkin/hlbudget/
    ├── repository/AccountColorMigrationIT.kt        # new: V5 → V6 back-fill
    ├── service/AccountServiceTest.kt                # colour on create, rename keeps it
    ├── service/TotalsServiceTest.kt                 # byAccount range rules and sums
    └── controller/                                  # AccountControllerIT, TotalsIT, TotalsPerformanceIT,
                                                     # OwnershipIT updated

hl-budget-frontend/
├── app/lib/api-schema.d.ts                          # regenerated
├── app/lib/definitions.ts                           # AccountTotals, AccountTotal aliases
├── app/lib/data.ts                                  # getAccountTotals; in loadSpendingsPage (R-009)
├── app/lib/phone.ts                                 # topCategories → topSlices
├── app/ui/spendings/breakdown-card.tsx              # from by-category-card.tsx, + kind (R-006)
├── app/ui/spendings/breakdown-table.tsx             # from category-table.tsx
├── app/ui/spendings/breakdown-pie.tsx               # from category-pie.tsx
├── app/ui/spendings/breakdown-kind.ts               # new: labels, test ids, storage keys per kind
├── app/ui/spendings/month-section.tsx               # second card: By account
├── app/ui/phone/breakdown.tsx                       # from category-breakdown.tsx, + kind (R-007)
├── app/ui/phone/phone-ui-provider.tsx               # view + 'by-account'; accountChart
├── app/ui/phone/month-year-picker.tsx               # third switch option
├── app/ui/phone/segmented.tsx                       # px-2 with 3+ options; wrap allowed (R-008)
├── app/ui/phone/phone-data.ts, phone-section.tsx    # accountTotals
├── app/ui/phone/phone-app.tsx                       # render By account
└── e2e/by-account.spec.ts, e2e/phone/by-account.spec.ts   # new; a11y specs cover the new views
```

**Structure Decision**: same two service repositories; no new modules. Renamed files keep their
tests (renamed alongside).

## Delivery order

1. **Branches**: `008-account-breakdown` from `origin/main` in both repos, plus this spec branch.
2. **Phase A, backend** (blocks the frontend's types):
   - A1: V6 migration and `AccountColorMigrationIT`; entity, row, model and response carry
     `color`.
   - A2: colour on create in `AccountService` with `colorCounts`; unit tests and
     `AccountControllerIT` (sequence, rename keeps, freed colour reused).
   - A3: `TotalsQueries.byAccount`, `TotalsService.byAccount`, `listAccountTotals` with OpenAPI
     docs and example; `TotalsIT` (sums, order, empty, range errors, isolation),
     `TotalsPerformanceIT`.
   - A4: export `contracts/api.yaml`, `./gradlew check`, PR, release, `hl-deploy-status`.
3. **Phase B, foundation**: regenerate types; `getAccountTotals` in `loadSpendingsPage`; rename
   the desktop and phone breakdown components behind `kind`, with By category behaving exactly as
   before (existing unit tests and e2e stay green).
4. **US1 desktop**: By account card in `MonthSection`; own storage key; unit tests; e2e desktop.
5. **US2 colours and labels**: verified through the shared components: account swatches, legend
   and slice colours match; hover/focus label names the account; e2e and axe.
6. **US3 phone**: third switch option, `accountChart`, `Breakdown kind="account"`; unit tests;
   phone e2e at 390 and 360 wide.
7. **Gates and deploy**: pnpm gates and `pnpm e2e`, frontend PR, release, `hl-deploy-status`,
   live check (quickstart §5), README update.

## Complexity Tracking

| Violation | Why Needed | Simpler Alternative Rejected Because |
|---|---|---|
| `GET /api/v1/account-totals` is not paginated (Principle VII: "Collection endpoints MUST be paginated") | It returns at most the user's account count (a handful), and the screen needs the whole set at once to draw shares and a total; same justification as `/category-totals` in the 003 plan | Paging would make the client loop for no gain. The response uses a named `items` field, so paging parameters can be added later as optional |
