# Implementation Plan: Fixed category colours, a working pie chart, and categories and accounts managed only in Settings

**Branch**: `007-fix-pie-csv-categories` | **Date**: 2026-10-31 | **Spec**: [spec.md](./spec.md)

**Input**: Feature specification from `/specs/007-fix-pie-csv-categories/spec.md`

## Summary

Three fixes. Each category gets a fixed colour, stored by the backend and used by every chart.
The donut shows a label on hover, focus or tap. Spendings and imports accept only existing
categories and accounts; unknown names are invalid fields and are never created.

Why the pie breaks today (research R-001): slices are coloured with CSS variables whose names are
built at run time (`var(--color-cat-${i + 1})`). Tailwind v4 drops theme variables it cannot see
used, so most slices point at a variable that does not exist and get no colour.

**Backend** (`hl-budget-backend`, v1, R-002 – R-008):
- Flyway V5: `categories.color` (`#rrggbb`, NOT NULL). Defaults are back-filled by name and custom
  categories by creation order.
- `CategoryPalette` holds 24 colours: 17 fixed for the defaults and 7 spare. A new category gets
  the least-used colour. Seeding writes the colours too.
- New required response fields `Category.color` and `CategoryTotal.color` (additive).
- `categoryName`/`accountName` on spending create/replace and on import rows now only match
  existing items. No match gives `422` with a `NotFound` violation per field, and nothing is
  written. The find-or-create inserts are removed.

**Frontend** (`hl-budget-frontend`, R-001, R-009 – R-013):
- Remove the `--color-cat-*` tokens, `category-colors.ts` and the phone's `CAT_COLORS`. Slices,
  legend and table swatches use the API's `color`.
- A shared `DonutChart` (desktop and phone): focusable slices, a centre label on hover, focus
  or tap, and other slices dimmed.
- `draftProblems` checks names against the user's categories and accounts ("Not one of your
  categories / accounts"). Because every surface calls it, this covers the entry row, the phone
  form, the review table and the phone Fix row.
- Server `NotFound` violations on names map to the same fields.
- No "create" suggestion. When the user has no accounts, a message links to Settings.

## Technical Context

**Language/Version**: Kotlin 2 / Java 21 (backend); TypeScript 5 strict (frontend).

**Primary Dependencies**: Spring Boot 4, JPA/Hibernate, Flyway, springdoc (backend); Next.js App
Router, React, Tailwind v4, `openapi-typescript` (frontend). No new dependencies.

**Storage**: PostgreSQL. One new column (`categories.color`) via V5.

**Testing**: JUnit 5 + Testcontainers Postgres, Kover (backend); Vitest + Testing Library,
Playwright with desktop and phone projects and axe (frontend).

**Target Platform**: homelab Kubernetes (Argo CD); browsers on desktop and phone.

**Project Type**: web application: two services in two repositories, with this spec repo
alongside.

**Performance Goals**: hover label within 0.5 s (SC-004; in practice the next frame). A 1,000-row
import must still pass the feature 004 performance IT: name lookup is one query per kind, not
one per row.

**Constraints**: the desktop layout stays as it is apart from the chart, the swatches and the
messages. No new API major version (see Complexity Tracking). Colours must be at least 3:1
against the card.

**Scale/Scope**: one user (a handful at most), 17 default categories plus a few custom, imports
of up to 1,000 rows.

All unknowns are resolved in [research.md](./research.md).

## Skills that drive the implementation

| Skill | Owns | Used in |
|---|---|---|
| `backend-developer` | V5 migration and IT, `CategoryPalette`, colour assignment, DTO fields and OpenAPI docs, name rule in `NamedItemResolver`, removing find-or-create, tests, `contracts/api.yaml` export, `./gradlew check` | Phase A |
| `frontend-developer` | Type regeneration, `DonutChart`, colour wiring, `draftProblems` lookups, messages, error mapping, no-accounts state, unit tests, e2e, pnpm gates | Phase B |
| `homelab:hl-deploy-status` | Verify both rollouts | Deploy |

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-checked after Phase 1 design.*

| Principle | Check | Result |
|---|---|---|
| I. Spec-driven | Spec 007 → plan → tasks before code | Pass |
| II. Separate services, contract-first | Colours and the name rule live in the backend (R-002, R-008). The frontend only displays colours and mirrors the rule to give early feedback; the server stays the authority (FR-028). Types are regenerated from the exported OpenAPI | Pass |
| III. Secure by default | Colour assignment and name lookups are scoped to the caller's `user_id`. Problem responses do not echo the rejected names | Pass |
| IV. Tests and gates | Migration IT, palette unit test, resolver/import/spending ITs updated, `DonutChart` and `draftProblems` unit tests, e2e for colours, label and unknown names on desktop and phone. Both gates must pass | Pass |
| V. Homelab-ready | New migration V5 only; V1–V4 untouched. Deploy goes through release → hl-helm → Argo CD | Pass |
| VI. Simplicity | No chart library and no new tables or services. The palette is a code constant | Pass |
| VII. Versioned API | Two new response fields (additive). Request shapes, status codes and problem types are unchanged. A business-rule change (names no longer create items) stays in v1; justified below | Pass with justification |

**Post-design re-check**: unchanged. The contract delta ([contracts/openapi-delta.yaml](./contracts/openapi-delta.yaml))
adds only response fields, descriptions and examples.

## Project Structure

### Documentation (this feature)

```text
specs/007-fix-pie-csv-categories/
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
│   └── V5__add_category_colors.sql                  # new (R-005)
├── src/main/kotlin/com/valyalkin/hlbudget/
│   ├── model/CategoryPalette.kt                     # new: 24 colours, nextColor(counts) (R-003, R-004)
│   ├── model/DefaultCategories.kt                   # NAMES → ALL (name, color) (R-006)
│   ├── model/Category.kt                            # + color
│   ├── repository/category/CategoryEntity.kt        # + color (not updatable)
│   ├── repository/category/CategoryRepository.kt    # + colour counts per user; names by user
│   ├── repository/CurrentUserRepository.kt          # insertDefaultCategories writes colours
│   ├── repository/NamedItemInserts.kt               # findOrCreate* removed (file deleted if empty)
│   ├── repository/spending/TotalsQueries.kt         # byCategory selects c.color
│   ├── service/CategoryService.kt                   # create assigns colour
│   ├── service/NamedItemResolver.kt                 # names only match; violations on miss (R-008)
│   ├── service/SpendingImportService.kt             # lookup before insert; per-row violations
│   ├── dto/category/CategoryResponse.kt             # + color
│   ├── dto/totals/TotalsResponses.kt                # CategoryTotalResponse + color
│   ├── dto/spending/SpendingRequest.kt              # name field descriptions
│   └── dto/spendingimport/SpendingImportRequest.kt  # name field descriptions
└── src/test/kotlin/com/valyalkin/hlbudget/
    ├── model/CategoryPaletteTest.kt                 # new: 24 distinct, ≥3:1, defaults distinct, assignment order
    ├── repository/CategoryColorMigrationIT.kt       # new: V4 → V5 back-fill
    └── controller/                                  # CategoryControllerIT, DefaultCategoriesIT, TotalsIT,
                                                     # SpendingByNameIT, SpendingImportsIT updated

hl-budget-frontend/
├── app/globals.css                                  # --color-cat-* removed
├── app/lib/api-schema.d.ts                          # regenerated
├── app/lib/draft.ts                                 # draftProblems(draft, today, lookups) (R-009)
├── app/lib/messages.ts                              # unknownCategory, unknownAccount, noAccounts
├── app/lib/actions.ts                               # ids when matched; NotFound → field mapping
├── app/lib/phone.ts                                 # CAT_COLORS removed; slices use item colour; lookups
├── app/lib/data.ts                                  # category totals carry color
├── app/ui/charts/donut-chart.tsx                    # new: shared donut with label (R-012)
├── app/ui/spendings/category-pie.tsx                # uses DonutChart
├── app/ui/spendings/category-colors.ts              # deleted
├── app/ui/spendings/category-table.tsx              # swatch (R-013)
├── app/ui/spendings/entry-row.tsx                   # lookups; no-accounts state (R-011)
├── app/ui/spendings/combobox.tsx                    # createHint ("Create “x”") removed (R-010)
├── app/ui/phone/category-breakdown.tsx              # uses DonutChart
├── app/ui/phone/spending-form.tsx                   # lookups; no-accounts state
├── app/ui/phone/suggest-input.tsx                   # no create option
├── app/ui/import/review-stage.tsx                   # lookups into draftProblems
├── app/ui/settings/accounts-panel.tsx               # empty-state text
└── e2e/                                             # by-category, add-spending, import-csv, settings, a11y and phone/* specs updated
```

**Structure Decision**: same two service repositories as features 003–006, with no new modules
apart from the shared `app/ui/charts/` folder, because the donut is now used by two screens.

## Delivery order

1. **Branches**: `007-fix-pie-csv-categories` from `origin/main` in both repos, plus this spec
   branch.
2. **Phase A, backend** (blocks the frontend's types):
   - A1: `CategoryPalette` and its unit test; `DefaultCategories.ALL`.
   - A2: V5 migration and its IT; entity, model and seeding with colours.
   - A3: colour assignment on create; `color` on `Category` and `CategoryTotal`; ITs.
   - A4: name rule in `NamedItemResolver`; import lookup with per-row violations; remove
     find-or-create; update `SpendingByNameIT` and `SpendingImportsIT` (including the
     1,000-row performance IT).
   - A5: OpenAPI descriptions and examples, export `contracts/api.yaml`, `./gradlew check`,
     PR, release, `hl-deploy-status`.
3. **Phase B, foundation**: regenerate types; `CategoryItem.color` through `data.ts` and
   `phone.ts`; messages.
4. **US1 colours**: remove the tokens and helpers; slices, legend and table swatches use
   `color`; unit tests; e2e on both projects.
5. **US2 label**: `DonutChart` with hover, focus and tap; wire into desktop and phone; unit tests
   for the state transitions; e2e and axe.
6. **US3 manual entry**: `draftProblems` lookups; entry row and phone form; no create option;
   `actions.ts` sends ids and maps `NotFound`; no-accounts state; accounts panel text; tests.
7. **US4 import**: review table and phone Fix row pass lookups; per-row server `NotFound`
   mapping; tests and e2e.
8. **Gates and deploy**: pnpm gates and `pnpm e2e`, frontend PR, release, `hl-deploy-status`,
   live check (quickstart §5), README update.

US1 and US2 need A3's types. US3 and US4 need A4 deployed to test against the real server, but
their local validation can start as soon as the foundation is in.

## Complexity Tracking

| Violation | Why Needed | Simpler Alternative Rejected Because |
|---|---|---|
| A business-rule change in v1: `categoryName`/`accountName` used to find-or-create and now find-or-refuse (`201` → `422` for an unknown name) | The spec requires that nothing outside Settings creates categories or accounts (FR-020, FR-026), and the server must enforce it (FR-028) | `/api/v2/spendings` and `/api/v2/spending-imports` would duplicate two endpoints and their tests, and keep v1 alive, only to preserve a behaviour the product now forbids. The shapes, status-code set and problem types do not change: `422` with `NotFound` violations is already documented for both endpoints. The frontend is the only client, it ships in the same feature, and the short window between the two deploys only affects typed new names (R-014). Removing the name fields entirely would be the breaking change that does need v2 |
