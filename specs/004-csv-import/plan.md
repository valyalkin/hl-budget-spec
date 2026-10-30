# Implementation Plan: Import Spendings from CSV

**Branch**: `004-csv-import` | **Date**: 2026-10-29 | **Spec**: [spec.md](./spec.md)

**Input**: Feature specification from `/specs/004-csv-import/spec.md`

## Summary

A signed-in user imports many spendings at once from a CSV file. The browser reads and parses
the file (it is never uploaded). The user checks and completes the rows in a review table that
reuses the Spendings table's calendar, autofill and keyboard entry. Then one request adds all
rows, or none.

**Backend** (`hl-budget-backend`, additive to v1, R-006, R-007):
- A new resource: `POST /api/v1/spending-imports` creates 1–1,000 spendings in one transaction.
  Categories and accounts are given by name and found or created once per name. All row errors
  are reported with JSON Pointers per row. An `Idempotency-Key` makes retries safe.
- `GET /api/v1/spending-imports/{id}` reads the import receipt back.
- One new table `spending_imports` (Flyway V4). It holds counts, dates, the total and the key,
  and nothing from the file.

**Frontend** (`hl-budget-frontend`, R-001 – R-005, R-008 – R-013):
- "Import CSV" button above the month total card.
- A `<dialog>` with two stages: pick (drop or choose a file, download the sample) and review
  (editable rows, highlighted problems, add or cancel).
- In-house CSV parsing and normalisation in `app/lib/` (day-first dates, amounts as decimal
  strings).
- One new Server Action, `importSpendings`.
- A static `public/sample-spendings.csv`.

There are no new runtime dependencies, environment variables or services.

## Technical Context

**Language/Version**:
- Frontend: TypeScript 5.9 (strict), Node 22, React 19.
- Backend: Kotlin 2.3 on Java 21.

Both are unchanged.

**Primary Dependencies**:
- Frontend: Next.js 16 (App Router, Cache Components, React Compiler), Tailwind 4, Zod 4,
  `openapi-typescript`. **No new dependency**: the CSV parser is in-house (R-002).
- Backend: Spring Boot 4.1, JDBC `JdbcClient` for batch inserts, springdoc 3.1. No new
  dependency.

**Storage**: PostgreSQL 17.
- One additive migration, `V4__create_spending_imports.sql` (data-model.md).
- Spendings, categories and accounts are unchanged.

**Testing**:
- Backend: JUnit 5, MockK, Testcontainers ITs (including a 1,000-row performance IT), Kover
  80/75 overall and 75 per package.
- Frontend: Vitest + Testing Library + MSW; Playwright `e2e/import-csv.spec.ts` plus axe scans of
  both dialog stages (R-014).

**Target Platform**: Evergreen desktop browsers (usable narrower, with sideways scrolling).
Homelab k3s via `charts/apps/hl-budget`, unchanged.

**Project Type**: Two-service web application (frontend + backend API).

**Performance Goals**:
- A 1,000-row file is in the review in under 3 s (SC-005); parsing is local.
- Editing updates highlights and totals at once.
- A 1,000-row import commits in under 2 s in an IT, which keeps SC-001 (a clean 30 rows in under
  1 minute) easy.

**Constraints**:
- No float money: amounts are decimal strings and integer cents in the UI (R-005); `numeric` in
  SQL.
- The file never leaves the browser and is never logged (FR-051).
- Tokens stay server-side.
- Additive v1 changes only.
- CSP unchanged: the sample is a same-origin static file.
- Files up to 1 MB and 1,000 rows.

**Scale/Scope**:
- Backend: 1 table, 2 operations, about 6 new classes.
- Frontend: about 6 new components, 2 new lib modules, 1 action and 1 static file.

All unknowns are resolved in [research.md](./research.md) (R-001 … R-015). No
`NEEDS CLARIFICATION` remains.

## Skills that drive the implementation

| Skill | Owns | Used in |
|---|---|---|
| `backend-developer` | Migration V4, `SpendingImportController` (full `@ApiResponse` sets and examples), DTOs, `SpendingImportService`, batch inserts, pointer-prefixed rules, tests, `contracts/api.yaml` export, `./gradlew check` | Phase A |
| `frontend-developer` | Lib modules, Server Action, components, sample file, tests, e2e, pnpm gates | Phase B |
| `homelab:hl-deploy-status` | Verify both rollouts | Deploy |

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

| Principle | Requirement | Plan | Status |
|---|---|---|---|
| I. Spec-driven, foundation-first | Spec → plan → tasks; independently deployable increment | Spec clarified (3 answers). The backend ships first and is inert until the frontend uses it | PASS |
| II. Separate services, contract-first | Backend owns data and rules; frontend has no business rules; generated types; RFC 9457 | All-or-nothing, name matching and creation, date/amount/currency rules and idempotency live in the backend. The frontend parses the file (input handling, R-001) and mirrors validation for instant feedback, as `draft.ts` already does; the server re-checks everything. Two reading rules live only in the browser because the server never sees the raw file: dates with the year last are read day first (R-004, clarification Q2), and a negative amount is read as a positive spending (R-005). Both only turn file text into draft values the user then reviews and can change, so they are input interpretation, not stored business rules; every value is validated again on the server. One Server Action, one API call. Types regenerated from `contracts/api.yaml` | PASS |
| III. Secure by default | No tokens in the browser; per-user scoping; noindex | The action re-checks the session; the token stays server-side. Every import row and name lookup filters on the caller's `user_id`, with an ownership IT. The file is not uploaded or logged. No new route | PASS |
| IV. Tests and quality gates | Tests ship with code; Playwright per new flow; gates green | R-014: unit tests for parser, mapping, dates, amounts, the sample file, validation and action; component tests; `e2e/import-csv.spec.ts`; axe; backend unit tests, ITs and a performance IT | PASS |
| V. Homelab-ready operations | Container contract; Flyway immutable; deploy via hl-helm | One new migration (V4), additive. No env, secret or chart change. Existing release workflows | PASS |
| VI. Simplicity at personal scale | No new services/caches/deps; justify additions | No new dependency or service. One small table, justified in R-006 (atomic + idempotent + resource for `Location`); no link column to spendings until a spec needs undo | PASS |
| VII. Versioned, consistent API | `/api/v1`, plural kebab-case nouns, no verbs, precise codes, Money as decimal string, validated with JSON Pointers, idempotency for money-moving creates, documented with examples | `POST /api/v1/spending-imports` (201 + `Location`, 200 replay, 400/401/422/500/503) and `GET /api/v1/spending-imports/{id}` (200/400/401/404/500/503). `Money` in rows, `MoneyTotal` in the receipt. Errors point at `#/spendings/{i}/…`. `Idempotency-Key` supported. Examples for every response | PASS |
| Tech & platform constraints | Listed stack; no float money | Stack unchanged. R-005 replaces the design prototype's `parseFloat` with decimal-string handling | PASS |

**Contract compatibility statement**: **backward compatible**, v1.
- Added: `createSpendingImport` and `getSpendingImport`; the schemas `SpendingImportRequest`,
  `SpendingImportRow` and `SpendingImport`; the response `SpendingImportValidationFailed`.
- Changed or removed: nothing.

**Post-design re-check (after Phase 1)**: PASS.
- [data-model.md](./data-model.md) adds one table with no change to existing ones.
- [contracts/openapi.yaml](./contracts/openapi.yaml) holds only the additions above.
- No complexity deviations are needed.

## Project Structure

### Documentation (this feature)

```text
specs/004-csv-import/
├── plan.md              # This file
├── research.md          # Phase 0: decisions R-001 … R-015
├── data-model.md        # Phase 1: spending_imports, client import state, parsing pipeline
├── quickstart.md        # Phase 1: gates, backend checks, browser journey, deploy
├── contracts/
│   └── openapi.yaml     # 2 new operations, 3 schemas, 1 response, examples
├── checklists/requirements.md
└── tasks.md             # Phase 2 (/speckit-tasks; not created here)
```

### Source Code

New files are marked `+` and changed files `~`.

**`hl-budget-backend`** (Phase A)

```text
src/main/resources/db/migration/
└── V4__create_spending_imports.sql          +
src/main/kotlin/com/valyalkin/hlbudget/
├── controller/SpendingImportController.kt   + POST /spending-imports, GET /{id}
├── dto/spendingimport/
│   ├── SpendingImportRequest.kt             + rows: @Size(1..1000) @Valid; row DTO
│   ├── SpendingImportResponse.kt            +
│   └── SpendingImportMappings.kt            +
├── model/SpendingImport.kt                  + receipt; ImportCommand, ImportRow, CreateImportResult
├── repository/spendingimport/
│   ├── SpendingImportEntity.kt              +
│   ├── SpendingImportRepository.kt          + findByIdAndUserId, findByUserIdAndIdempotencyKey
│   └── SpendingImportInserts.kt             + insert unless key taken (ON CONFLICT DO NOTHING)
├── repository/spending/SpendingInserts.kt   ~ batchInsert(rows) via JdbcClient batch
├── service/
│   ├── SpendingImportService.kt             + validate all → replay check → receipt → names once → batch insert
│   ├── SpendingRules.kt                     ~ violations(command, pointerPrefix = ""), importFingerprint(rows)
│   └── NamedItemResolver.kt                 ~ resolveNames(userId, distinct names) shared with single create
├── error/ (unchanged; 422 builder already lists every violation)
└── config/OpenApiConfig.kt, OpenApiBudgetComponents.kt  ~ tag "Spending imports", SpendingImportValidationFailed
src/test/kotlin/com/valyalkin/hlbudget/
├── service/SpendingImportServiceTest.kt, SpendingRulesTest.kt ~        + / ~
└── controller/SpendingImportsIT.kt, SpendingImportOwnershipIT.kt,
               SpendingImportPerformanceIT.kt                          +
contracts/api.yaml                           ~ regenerated
README.md                                    ~ new endpoints
```

**`hl-budget-frontend`** (Phase B)

```text
contracts/api.yaml, app/lib/api-schema.d.ts  ~ regenerated
public/sample-spendings.csv                  + R-011
app/
├── spendings/page.tsx                       ~ wrap content in <ImportProvider> outside <Suspense>
├── lib/
│   ├── csv.ts                               + parseCsv (RFC 4180, R-002)
│   ├── csv-import.ts                        + mapColumns, parseImportDate, parseImportAmount,
│   │                                          normaliseName, rowsFromCsv, file checks (R-003–R-005, R-012)
│   ├── draft.ts                             ~ draftProblems (all problems); draftProblem = first of them
│   ├── actions.ts                           ~ importSpendings (R-013)
│   ├── messages.ts                          ~ import messages: notCsv, tooLarge, unreadable, noRows,
│   │                                          added(n, file), needAttention(n)
│   └── definitions.ts                       ~ SpendingImport, SpendingImportRow aliases
└── ui/
    ├── spendings/
    │   ├── month-section.tsx                ~ ImportCsvButton above MonthTotalCard
    │   ├── spendings-table.tsx              ~ report "row open" to the provider (button disabled)
    │   ├── entry-row.tsx                    ~ keyboard helpers moved to row-keys.ts
    │   └── row-keys.ts                      + arrowStep, nextField, merchant fill-in (shared)
    └── import/
        ├── import-provider.tsx              + 'use client': context, stage, rows, key, notice
        ├── import-csv-button.tsx            + 'use client': opens with page lookups
        ├── import-dialog.tsx                + 'use client': <dialog>, Escape/cancel rules, stages
        ├── pick-stage.tsx                   + drop area, file input, sample link, refusals
        ├── review-stage.tsx                 + badge, table, footer, add/cancel
        ├── import-row.tsx                   + DatePicker + Combobox ×3 + amount + remove
        └── import-notice.tsx                + role=status "Added N spendings from …"
   (each with a colocated *.test.tsx; csv.test.ts, csv-import.test.ts incl. the sample file)
e2e/
├── import-csv.spec.ts                       + US1–US4
├── a11y.spec.ts                             ~ axe on pick and review stages
└── fixtures/
    ├── make-csv.mjs                         + N-row generator (SC-005 check)
    └── *.csv                                + clean, gaps, not-csv, header-only fixtures
mocks/handlers.ts                            ~ MSW for spending-imports
```

**Structure Decision**: The existing one-service-per-repo layout stays.
- Backend: the `backend-developer` package roles, with a feature subpackage in `dto/` and
  `repository/` as in 002/003.
- Frontend: a new feature folder `app/ui/import/`, the parsing in `app/lib/`, and the write in
  `actions.ts`.

There is no new route: the import is a dialog on `/spendings`.

## Delivery order

1. **Branches**: `004-csv-import` from `origin/main` in both repos (R-015).
2. **Phase A, backend** (blocks the frontend's types):
   - A1: migration V4, entity, repository, receipt inserts.
   - A2: pointer-prefixed `SpendingRules`; shared name resolution; batch insert.
   - A3: `SpendingImportService` (validate, replay, receipt, names, rows) with unit tests.
   - A4: controller, OpenAPI docs and examples, ITs (contract, ownership, performance).
   - A5: export `contracts/api.yaml`, `./gradlew check`, PR, release, `hl-deploy-status`.
3. **Phase B, foundation**:
   - regenerate types;
   - `csv.ts`, `csv-import.ts` and the sample file with their tests;
   - `draftProblems`;
   - messages.
4. **US1** clean import: provider, button, dialog pick stage (drop + choose), review read-only
   path, `importSpendings`, notice, navigation to the latest month.
5. **US2** fix rows: `row-keys.ts` extraction (003's entry-row tests must stay green),
   `ImportRow` editing with autofill and calendar, highlighting, badge, totals, remove,
   server-error mapping.
6. **US3** sample download link and its test.
7. **US4** cancel paths and Escape ordering.
8. **Polish**: refusal messages, disabled-while-row-open, axe, e2e, SC-005 check with 1,000 rows.
9. **Gates and deploy**: pnpm gates and `pnpm e2e`, frontend PR, release, `hl-deploy-status`,
   then the live check and the README update (quickstart §4).

US1 needs A5's types. US2 needs US1. US3 and US4 need only the dialog from US1.

## Complexity Tracking

No constitution violations. The one addition (the `spending_imports` table) is required by
Principle VII (a resource for `201 Location`, idempotent money-moving create) and FR-033
(atomic). Its justification and the rejected alternatives are in research R-006.
