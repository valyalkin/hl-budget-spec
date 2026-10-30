---

description: "Task list for 004-csv-import"
---

# Tasks: Import Spendings from CSV

**Input**: Design documents from `/specs/004-csv-import/`

**Prerequisites**: plan.md, spec.md, research.md, data-model.md, contracts/openapi.yaml,
quickstart.md

**Tests**: REQUIRED. Constitution Principle IV says tests ship with the code and every new
user-visible flow gets a Playwright spec. Research R-014 lists them. Within each phase, write
the tests first and see them fail before writing the code that satisfies them.

**Organization**: Tasks are grouped by user story (spec.md US1–US4), so each story can be built
and checked on its own. The backend is needed only by US1 (adding), so it lives in that phase
and can run in parallel with the frontend foundation.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Can run in parallel (different files, no dependency on an unfinished task).
- **[Story]**: The user story the task belongs to (US1 … US4).
- Every task ends with `· skill: <name>`. Load that skill before starting the task; its
  definition of done is part of the task's done. Where a skill and the constitution disagree,
  the constitution wins.

## Path Conventions

All paths are relative to the spec repo root `/Users/valyalkin/Documents/Projects/hl-budget-spec/`.

- `hl-budget-backend/`: the git worktree of `valyalkin/hl-budget-backend`, on branch
  `004-csv-import` (T001).
  - Main code: `hl-budget-backend/src/main/kotlin/com/valyalkin/hlbudget/`, written below as
    `…/main/`.
  - Tests: `hl-budget-backend/src/test/kotlin/com/valyalkin/hlbudget/`, written below as
    `…/test/`.
- `hl-budget-frontend/`: the git worktree of `valyalkin/hl-budget-frontend`, on branch
  `004-csv-import` (T001). Written below as `fe/`.
- **Commits**: commit inside the service repos using Conventional Commits, after each task or
  logical group.
- **Merging**: merging to `main` in a service repo triggers a release. Do it only after the gate
  is green and the user agrees.

Shared rules that several tasks reference:

- **Regenerate types**: run `./gradlew check` in `hl-budget-backend/`; `OpenApiExportIT` rewrites
  `hl-budget-backend/contracts/api.yaml`. Then run
  `cp hl-budget-backend/contracts/api.yaml hl-budget-frontend/contracts/api.yaml`, and in `fe/`
  run `pnpm generate:api`. Commit both frontend files.
- **Money**:
  - Backend: `BigDecimal`/`numeric` only.
  - Frontend: amounts are decimal strings. Convert to integer cents only with `toCents`, and back
    with `centsToAmount`/`formatCents` from `fe/app/lib/utils.ts`. Never use `parseFloat`,
    `Number(…)` or `Math.round` on money (research R-005).
- **Messages**: every user-facing string comes from `fe/app/lib/messages.ts`, worded exactly as
  in spec.md.
- **Server Action rules** (as in 003):
  1. Re-check the session; with no session, return `{ status: 'signed-out' }`.
  2. Validate the input with Zod.
  3. Make exactly one `authedFetch` call.
  4. On success, call `refresh()`.
  5. Never throw to the client.

  A client caller that gets `signed-out` calls `window.location.assign('/')`.
- **Client components**: `'use client'` only on the files plan.md marks. No `useMemo`,
  `useCallback` or `React.memo` (the React Compiler is on).
- **Not yours = not found** (backend): every new query filters on the caller's `userId`.
- **The file never leaves the browser** (FR-051): no task may send raw file text to a Server
  Action, the API or a log. Only reviewed rows are sent.

---

## Phase 1: Setup

**Purpose**: Fresh feature branches with green gates.

- [X] T001 In both `hl-budget-backend/` and `hl-budget-frontend/`:
  1. Run `git fetch origin` and `git switch -c 004-csv-import origin/main`.
  2. Before any change, confirm the gates are green: `./gradlew check` in the backend, and
     `pnpm install && pnpm typecheck && pnpm lint && pnpm test` in the frontend.

  The spec repo is already on `004-csv-import`. Ask the user before committing
  `specs/004-csv-import/` there · skill: none

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: Turning a file into review rows, and validating rows. Every story needs these.
Frontend only; the backend track (T010–T022) can start in parallel.

**⚠️ CRITICAL**: No frontend user-story work starts until this phase is complete.

### Tests first (see them fail)

- [X] T002 [P] Create `fe/app/lib/csv.test.ts` for `parseCsv(text: string): string[][]`
  (research R-002). Cases:
  - `a,b\n1,2` → `[["a","b"],["1","2"]]`.
  - The `\r\n` and `\r` line endings.
  - A quoted value with a comma: `"Lakeshore Fuel, Hwy 7"`.
  - A quoted value with an escaped quote: `"5"" pipe"` → `5" pipe`.
  - A quoted value with a line break inside.
  - Blank and whitespace-only lines are skipped.
  - A leading `﻿` is stripped.
  - A trailing newline adds no empty row.
  - Empty cells are kept: `a,,c` → `["a","","c"]`.
  - `''` → `[]`.

  · skill: frontend-developer
- [X] T003 [P] Create `fe/app/lib/csv-import.test.ts` for the functions of research R-003 to
  R-005 and R-012:
  - **`mapColumns(header: string[])`** → `{ date, merchant, category, account, amount }`
    column indexes, or `-1` for each missing one:
    - `Date,Description,Category,Account,Amount` → 0, 1, 2, 3, 4.
    - Reordered headers in any case (`AMOUNT,Payee,Posted Date`) are found.
    - `Account name` goes to account, not merchant.
    - A header that matches nothing is ignored.
    - When fewer than 2 fields match, it returns `null` (meaning no header).
  - **`parseImportDate(text)`** → `YYYY-MM-DD` or `''`:
    - `2026-10-28`, `2026/10/28` and `2026.10.28` → `2026-10-28`.
    - `28/10/2026` → `2026-10-28`.
    - `03/04/2026` → `2026-04-03` (day first, always).
    - `3-4-2026` → `2026-04-03`.
    - `10/27/2026` → `''` (month 27).
    - `2026-02-30` → `''`.
    - `2026-10-28 14:03` and `2026-10-28T14:03:00` → `2026-10-28`.
    - `yesterday` and `''` → `''`.
    - A future date is returned as read.
  - **`parseImportAmount(text)`** → canonical decimal string or `''`:
    - `12.40` → `12.40`.
    - `-18.90`, `18.90-` and `(18.90)` → `18.90`.
    - `$1,041.18` and `"1,041.18"` → `1041.18`.
    - `CAD 5` → `5.00`.
    - `6.755` → `6.76` (half up, on the string).
    - `0`, `0.00`, `abc`, `''` and `1.2.3` → `''`.
    - Assert by inspection that no `parseFloat` or `Number(` is used on the value.
  - **`normaliseName(text)`**: trims and collapses inner whitespace runs to one space.
  - **`rowsFromCsv(text, { categories, accounts })`** → `ImportRow[]` (data-model.md
    "ImportRow": `key`, `draft`):
    - With a header, every data row is in file order.
    - A missing column leaves that field `''` in every row.
    - No header: the fixed order date, merchant, category, account, amount, and row 1 is data.
    - `visa` becomes the existing `Visa`, and an unknown name is kept as typed (FR-014).
    - Each row's `key` is unique and stable.
  - **`checkFile(file: { name, type, size })`** → `null` or a message key:
    - `.png` with type `image/png` → `notCsv`.
    - `data.CSV` with an empty type → ok.
    - `export.txt` with type `text/csv` → ok.
    - `application/vnd.ms-excel` with a `.csv` name → ok.
    - `size > 1_048_576` → `tooLarge`.
  - **`readImport(text, lookups)`** → `{ rows } | { error }`:
    - `''` and header-only → `noRows`.
    - 1,001 data rows → `tooLarge`.
    - 1,000 data rows → ok.

  · skill: frontend-developer
- [X] T004 [P] Extend `fe/app/lib/draft.test.ts` for
  `draftProblems(draft, today): Partial<Record<Field, string>>`:
  - A valid draft → `{}`.
  - An empty draft → all five fields with `Pick a date.`, `Add a merchant name.`, `Choose or type
    a category.`, `Choose or type an account.` and `Enter an amount greater than zero.`.
  - A future date → `date: 'Pick a date that is today or earlier.'`.
  - A 256-character merchant, and a 51-character category or account, are flagged.

  Also check that the existing `draftProblem` still returns the first problem in field order
  (003 behaviour unchanged) · skill: frontend-developer
- [X] T005 [P] Extend `fe/app/lib/messages.test.ts` with the import strings, exactly as spec.md
  words them:
  - `MESSAGES.notCsv` = `That doesn’t look like a CSV file. Choose a .csv export.`
  - `MESSAGES.unreadable` = `The file couldn’t be read. Try again.`
  - `MESSAGES.noRows` = `No spendings found in that file. Check that it is a CSV with a header
    row and at least one data row.`
  - `MESSAGES.tooLarge` = `That file is too large. Import at most 1 MB and 1,000 rows at a time.`
  - `needAttention(1)` = `1 row needs attention`; `needAttention(4)` = `4 rows need attention`.
  - `added(12, 'oct.csv')` = `Added 12 spendings from oct.csv`; `added(1, …)` uses `1 spending`.
  - `addLabel(0)` = `Add spendings`, `addLabel(1)` = `Add 1 spending`, `addLabel(10)` =
    `Add 10 spendings`.
  - `rowCount(10)` = `10 rows`.
  - `MESSAGES.alreadyImported` = `These spendings were already added.`

  · skill: frontend-developer

### Implementation

- [X] T006 [P] Create `fe/app/lib/csv.ts` with `parseCsv` (a single-pass state machine, comma
  only, no dependency) and make T002 green · skill: frontend-developer
- [X] T007 Create `fe/app/lib/csv-import.ts` with `mapColumns`, `parseImportDate`,
  `parseImportAmount`, `normaliseName`, `rowsFromCsv`, `checkFile`, `readImport` and the
  exported constants `MAX_FILE_BYTES = 1_048_576` and `MAX_ROWS = 1000`. Follow these rules:
  - **Column patterns**: as in the research R-003 table, checked in the order date, category,
    account, amount, merchant.
  - **Dates**: R-004. Cut at the first space or `T`; check impossible dates with `isValidDate`
    from `utils.ts`.
  - **Amounts**: R-005, string-only rounding through `toCents`/`centsToAmount`.
  - **Name matching**: `exactMatch` from `utils.ts`.

  Export the type `ImportRow = { key: string; draft: Draft }`. Make T003 green
  · skill: frontend-developer
- [X] T008 [P] In `fe/app/lib/draft.ts`, add `draftProblems`, which collects every problem with
  the same rules and messages as `draftProblem`. Rewrite `draftProblem` as the first entry of
  `draftProblems` in `FIELDS` order. Make T004 green; `fe/app/ui/spendings/entry-row.test.tsx`
  must stay green · skill: frontend-developer
- [X] T009 [P] In `fe/app/lib/messages.ts`, add `notCsv`, `unreadable`, `noRows`, `tooLarge` and
  `alreadyImported` to `MESSAGES`, plus the helpers `needAttention`, `added`, `addLabel` and `rowCount` (using
  `plural`). Make T005 green · skill: frontend-developer

**Checkpoint**: `pnpm typecheck && pnpm lint && pnpm test` is green. A file's text can now be
turned into validated draft rows.

---

## Phase 3: User Story 1 — Import a clean CSV file (Priority: P1) 🎯 MVP

**Goal**: The user clicks "Import CSV", drops or chooses a clean file, sees "All rows look good",
clicks "Add N spendings" and lands on the latest imported month with "Added N spendings from
[file]". All rows are saved in one all-or-nothing request.

**Independent Test**: spec.md US1. Import a 5-row clean CSV spread over September and October.
The review shows 5 rows and "All rows look good". After clicking "Add 5 spendings", all 5 are in
the right months on screen and on the server, with the file's exact values.

### Backend tests (write first, see them fail)

- [X] T010 [P] [US1] Extend `…/test/service/SpendingRulesTest.kt` (create it if missing):
  - `violations(command, pointerPrefix = "#/spendings/3")` reports the pointer
    `#/spendings/3/date` and the field `spendings[3].date` for a future date.
  - It reports `#/spendings/3/amount/amount` for a zero amount.
  - It reports `#/spendings/3/amount/currency` for another currency.
  - The default prefix keeps 003's `#/date`.

  For `importFingerprint(rows)`:
  - The same rows with names in another case and `"84.3"` versus `"84.30"` → the same hash.
  - Rows in a different order → a different hash.

  · skill: backend-developer
- [X] T011 [P] [US1] Write `…/test/service/SpendingImportServiceTest.kt` (MockK for
  repositories, a fixed `Clock`). Cases:
  1. **Valid rows**: names are resolved once per `lower(trim(name))`. `Visa`, `visa ` and `VISA`
     make a single `findOrCreateAccount` call. One `batchInsert` covers all rows. The receipt
     has `spendingCount`, `firstDate`, `lastDate` and the `total` sum.
  2. **Invalid row**: any invalid row throws `ValidationException` listing every violation of
     every row, and makes no insert or name call.
  3. **Same key and fingerprint**: returns the existing receipt with `replayed = true` and
     inserts nothing.
  4. **Same key, different fingerprint**: throws `UnprocessableEntityException` with
     `reason = idempotency-key-reused`.
  5. **Lost key race**: `insertUnlessKeyTaken` returns false → the winner is read and replayed.

  · skill: backend-developer
- [X] T012 [P] [US1] Write `…/test/controller/SpendingImportsIT.kt` (Testcontainers, real JWTs
  as in `SpendingControllerIT`), following contracts/openapi.yaml.
  - **The `twoRows` example**:
    - `201`, with `Location: /api/v1/spending-imports/{id}`.
    - The body matches `SpendingImport` (`spendingCount: 2`, `firstDate: "2026-09-30"`,
      `lastDate: "2026-10-28"`, `total: {amount: "73.60", currency: "CAD"}`).
    - Both spendings appear in `GET /api/v1/spendings`.
    - `car` reuses the default `Car`.
    - A new account `Amex` is created once.
  - **Replay** with the same `Idempotency-Key` → `200`, `Idempotent-Replayed: true`, the same
    `id`, still 2 spendings.
  - **Same key, other rows** → `422`, `details.reason = idempotency-key-reused`.
  - **Invalid rows**: 3 rows, row 1 dated tomorrow and row 2 with amount `"0"`:
    - `422` with pointers `#/spendings/1/date` and `#/spendings/2/amount/amount`.
    - The row counts of `spendings`, `categories`, `accounts` and `spending_imports` are
      unchanged, even when rows 0 and 2 name a new category.
  - **Row counts**: an empty `spendings` → `422` on `#/spendings`; 1,001 rows → `422` on
    `#/spendings`.
  - **Bad fields**: a blank `merchant` → `#/spendings/0/merchant`; a 51-character
    `categoryName` → `#/spendings/0/categoryName`.
  - **Malformed JSON** → `400`.
  - **No token** → `401`.
  - **`GET` the `Location`** → `200` with the same summary.

  · skill: backend-developer
- [X] T013 [P] [US1] Extend `…/test/controller/OwnershipIT.kt`:
  - User B's `GET /api/v1/spending-imports/{A's id}` → `404`.
  - User B importing `accountName: "Visa"` creates B's own `Visa` and never uses A's.
  - User B replaying A's `Idempotency-Key` creates B's import; keys are per user.

  · skill: backend-developer

### Backend implementation

- [X] T014 [US1] Create `hl-budget-backend/src/main/resources/db/migration/V4__create_spending_imports.sql`
  exactly as in data-model.md:
  - **Columns**:
    - `id uuid NOT NULL DEFAULT gen_random_uuid()` (PK `pk_spending_imports`);
    - `user_id uuid NOT NULL`, FK `fk_spending_imports_user` → `users (id) ON DELETE CASCADE`;
    - `idempotency_key varchar(255) NULL`;
    - `idempotency_fingerprint char(64) NULL`;
    - `spending_count integer NOT NULL`;
    - `first_date date NOT NULL`;
    - `last_date date NOT NULL`;
    - `total numeric(14,2) NOT NULL`;
    - `created_at timestamptz NOT NULL DEFAULT now()`.
  - **Constraints**:
    - `ck_spending_imports_count CHECK (spending_count BETWEEN 1 AND 1000)`;
    - `ck_spending_imports_dates CHECK (first_date <= last_date)`;
    - `ck_spending_imports_total CHECK (total > 0)`;
    - `ck_spending_imports_idempotency_pair CHECK ((idempotency_key IS NULL) = (idempotency_fingerprint IS NULL))`.
  - **Index**: `CREATE UNIQUE INDEX ux_spending_imports_user_key ON spending_imports (user_id, idempotency_key) WHERE idempotency_key IS NOT NULL`.
  - Add a header comment pointing at `specs/004-csv-import/data-model.md`.

  `SchemaValidationIT` must stay green · skill: backend-developer
- [X] T015 [P] [US1] Create the persistence layer for imports:
  - **`…/main/repository/spendingimport/SpendingImportEntity.kt`**: JPA, mapped to
    `spending_imports`; `total` is a `BigDecimal`.
  - **`…/main/repository/spendingimport/SpendingImportRepository.kt`**:
    `findByIdAndUserId(id, userId)` and `findByUserIdAndIdempotencyKey(userId, key)`.
  - **`…/main/repository/spendingimport/SpendingImportInserts.kt`**:
    `insertUnlessKeyTaken(entity): Boolean`, using `JdbcClient` with
    `INSERT … ON CONFLICT (user_id, idempotency_key) WHERE idempotency_key IS NOT NULL DO NOTHING`.
    Mirror `SpendingInserts` and its comment on why.
  - **A model mapping** to `model/SpendingImport.kt`.

  · skill: backend-developer
- [X] T016 [P] [US1] In `…/main/repository/spending/SpendingInserts.kt`, add
  `batchInsert(spendings: List<SpendingEntity>)`. It uses one JDBC batch, with
  `idempotency_key`/`idempotency_fingerprint` `NULL`, and `version` 0. Assert that the update
  count equals the list size · skill: backend-developer
- [X] T017 [US1] In `…/main/service/SpendingRules.kt`:
  - **Pointer prefix**: give `violations(command)` an optional `pointerPrefix: String = "#"` and
    a matching `fieldPrefix`. Build pointers as `"$pointerPrefix/date"` and fields as
    `spendings[3].date` when prefixed, so 003 output is unchanged.
  - **`importFingerprint(rows: List<ImportRowCommand>)`**: SHA-256 over the rows in order, each
    `date|trim(merchant)|amount at scale 2|currency|lower(trim(categoryName))|lower(trim(accountName))`,
    joined with `\n` (data-model.md "Fingerprint").

  Make T010 green · skill: backend-developer
- [X] T018 [US1] In `…/main/service/NamedItemResolver.kt`, add
  `resolveNames(userId, categoryNames: Collection<String>, accountNames: Collection<String>, now): NameMaps`.
  - It de-duplicates by `lower(trim(name))` and calls `NamedItemInserts.findOrCreate…` once per
    distinct name.
  - It returns `Map<String (lowercased), Ref>` for each kind.

  Keep `resolve(...)` for single spendings unchanged; `SpendingByNameIT` stays green
  · skill: backend-developer
- [X] T019 [P] [US1] Create the DTOs and the model:
  - **`…/main/dto/spendingimport/SpendingImportRequest.kt`**: `@Schema(name = "SpendingImportRequest")`,
    with `spendings: List<@Valid SpendingImportRowRequest>` and
    `@field:Size(min = 1, max = 1000, message = "must have 1 to 1000 rows")`.
  - **`SpendingImportRowRequest`** (`@Schema(name = "SpendingImportRow")`): `date` (`@NotNull`);
    `merchant` (`@NotBlank`, `@Size(max = 255)`); `amount: MoneyDto` (`@Valid`, `@NotNull`);
    `categoryName` and `accountName` (`@NotBlank`, `@Size(max = 50)`). Use the descriptions and
    examples from contracts/openapi.yaml.
  - **`SpendingImportResponse.kt`** (`@Schema(name = "SpendingImport")`): `id`, `spendingCount`,
    `firstDate`, `lastDate`, `total: MoneyTotalDto`, `createdAt`.
  - **`SpendingImportMappings.kt`**: `toCommand()` and `toResponse()`.
  - **`…/main/model/SpendingImport.kt`**: `SpendingImport`, `ImportRowCommand` and
    `CreateImportResult(import, replayed)`.

  · skill: backend-developer
- [X] T020 [US1] Create `…/main/service/SpendingImportService.kt`, `@Transactional`, with
  `create(identity, rows, idempotencyKey): CreateImportResult` following data-model.md
  "Write: create an import". The steps:
  1. Resolve the user.
  2. Collect `rules.violations(row, "#/spendings/$i")` for every row; throw one
     `ValidationException` if any.
  3. With a key: compute the fingerprint, then look up an existing import → replay or
     `idempotency-key-reused`, worded `The Idempotency-Key was already used for a different import`.
  4. Build the receipt (`spendingCount`, `firstDate`, `lastDate`, `total` as a `BigDecimal` sum).
     If `insertUnlessKeyTaken` returns false, read the winner and replay.
  5. Call `resolveNames`.
  6. Call `batchInsert` with `createdAt = updatedAt = now`.

  Add `get(identity, id)` → `NotFoundException` with `resource = spending-import` when the import
  is missing or another user's. Make T011 green · skill: backend-developer
- [X] T021 [US1] Create `…/main/controller/SpendingImportController.kt` at
  `/api/v1/spending-imports`, tagged `Spending imports`.
  - **`POST`** (operationId `createSpendingImport`):
    - takes the optional `Idempotency-Key` header (`@Size(max = 255)`);
    - returns `201` + `Location`, or `200` + `Location` + `Idempotent-Replayed: true` on replay,
      using `ResponseEntity` as in `SpendingController.createSpending`.
  - **`GET /{importId}`** (operationId `getSpendingImport`).
  - **Documentation**: every response and example exactly as in
    `specs/004-csv-import/contracts/openapi.yaml`.
  - **Shared components**: in `…/main/config/OpenApiConfig.kt`, add the tag
    `Spending imports` ("Adding many spendings at once, all or nothing"). In
    `…/main/config/OpenApiBudgetComponents.kt`, add the `SpendingImportValidationFailed`
    response with its three examples.

  Make T012 and T013 green · skill: backend-developer
- [X] T022 [US1] Run `./gradlew check` (Kover floors 80/75 overall, 75 per package). Check that
  `git diff origin/main -- contracts/api.yaml` contains only what
  `specs/004-csv-import/contracts/openapi.yaml` describes. Then **regenerate types** (shared
  rule) · skill: backend-developer

### Frontend tests (write first, see them fail)

- [X] T023 [P] [US1] Extend `fe/app/lib/actions.test.ts` (MSW) for
  `importSpendings({ rows, idempotencyKey, today, currency })`:
  - **Success**:
    - exactly one `POST /api/v1/spending-imports`, with the `Idempotency-Key` header;
    - the body rows have trimmed names in `categoryName`/`accountName` and amounts from
      `parseAmountInput`;
    - returns `{ status: 'ok', month: '2026-10', count: 2 }` from `lastDate`;
    - `refresh` is called.
  - **Local checks**: a row that fails `draftProblems` → `{ status: 'error', rowErrors }`
    without any request.
  - **Bad input**: more than 1,000 rows, or a non-UUID key → an error without a request.
  - **`422` mapping**:
    - errors on `#/spendings/1/date` and `#/spendings/2/amount/amount` →
      `rowErrors: [{ row: 1, field: 'date', message: 'Pick a date that is today or earlier.' }, { row: 2, field: 'amount', message: 'Enter an amount greater than zero.' }]`;
    - `categoryName` maps to `category`, `accountName` to `account`.
  - **No session** → `signed-out`.
  - **Unreachable or 5xx** → `MESSAGES.unavailable`.
  - **`422` with `details.reason = idempotency-key-reused`** → `{ status: 'already-imported' }`
    (an earlier attempt with this key was saved), and `refresh` is called.

  · skill: frontend-developer
- [X] T024 [P] [US1] Create `fe/app/ui/import/import-provider.test.tsx` and
  `fe/app/ui/import/import-csv-button.test.tsx` (RTL):
  - The button reads "Import CSV" with the upload icon (`aria-hidden`).
  - Clicking it calls `open` with the page's categories, accounts, merchants, today and currency.
  - The provider exposes `stage === 'pick'` after `open`.
  - The provider renders `ImportNotice` (`role="status"`) when `notice` is set, and the notice
    persists when the children re-render.

  · skill: frontend-developer
- [X] T025 [P] [US1] Create `fe/app/ui/import/pick-stage.test.tsx` (RTL + user-event):
  - **Content** matches spec US1 #2: the heading "Import CSV", the explanation text, "Drop your
    CSV file here", the "Choose file" input with `accept=".csv,text/csv"`, the "What the file
    should contain" note, Cancel, and a close button labelled "Close import".
  - **Drag**: `dragover` sets the highlight (`data-dragging="true"`); `dragleave` clears it.
  - **Valid file**: dropping a valid File calls `onRows(fileName, rows)`.
  - **Choosing a file** behaves the same.
  - **Several files**: only the first is used.
  - **Refusals**: a `.png` shows `MESSAGES.notCsv` in a `role="alert"`, and so do the
    `tooLarge`, `noRows` and `unreadable` cases (`file.text()` rejects). None of them calls
    `onRows`.

  · skill: frontend-developer
- [X] T026 [P] [US1] Create `fe/app/ui/import/review-stage.test.tsx` (RTL) for the all-valid
  path:
  - The header reads `transactions.csv · 2 rows`.
  - It shows "All rows look good".
  - The footer reads `2 spendings · total $73.60`, summed with `toCents`.
  - The button reads "Add 2 spendings".
  - Clicking it calls `importSpendings` once with the rows and the session's key, and the button
    is disabled while pending (FR-034).
  - On `ok`, it calls `onAdded(month, count)`.
  - On `error` with no `rowErrors`, it shows the message and keeps the rows.
  - On `signed-out`, it calls `window.location.assign('/')`.

  · skill: frontend-developer
- [X] T027 [P] [US1] Create `fe/e2e/import-csv.spec.ts`. **Do not commit a static clean
  fixture**: its dates would go stale. The test builds the file at run time with a helper
  `cleanCsv(today)` in `fe/e2e/support.ts`:
  - 5 rows with the header `Date,Description,Category,Account,Amount`;
  - day-first dates: 2 in last month, 3 in the current month, none after today;
  - existing default categories and account `Visa`;
  - merchants carrying the e2e cleanup marker.

  It passes the result to `setInputFiles({ name: 'clean.csv', mimeType: 'text/csv', buffer })`.
  The US1 test:
  1. Open Import CSV and attach the generated file.
  2. Expect "All rows look good", then click "Add 5 spendings".
  3. Expect the URL month to be the latest date's month and the status text
     `Added 5 spendings from clean.csv`.
  4. Expect the rows in the table, and the other month's rows after switching months.
  5. Clean up through the UI helpers.

  · skill: frontend-developer

### Frontend implementation

- [X] T028 [US1] In `fe/app/lib/definitions.ts`, add aliases `SpendingImport` and
  `SpendingImportRow` from the regenerated schema. In `fe/mocks/handlers.ts`, add MSW handlers
  for `POST /api/v1/spending-imports` (201 with the contract example) and the `GET`
  · skill: frontend-developer
- [X] T029 [US1] In `fe/app/lib/actions.ts`, add `importSpendings` following the Server Action
  rules and research R-013:
  - **Zod**: `rows` is an array of 1 to 1,000 `{ key, draft }`; `idempotencyKey` is `z.uuid()`;
    `today` is `z.iso.date()`; `currency` is 3 capitals.
  - **Local checks**: `draftProblems` per row; any problem returns `rowErrors` without a request.
  - **Request**: map each row to `SpendingImportRow`; amounts from `parseAmountInput`.
  - **Responses**:
    - map `ApiError.errors` pointers `#/spendings/{i}/{field}` back to row and field;
    - map `field` to the spec message through `fieldFor`/`messageFor` in `messages.ts`;
    - extend `fieldFor` if needed, so `categoryName` → `category` and `accountName` →
      `account`;
    - a `422` whose `reason` is `idempotency-key-reused` → `refresh()` and
      `{ status: 'already-imported' }` (add it to this action's result type).

  Make T023 green · skill: frontend-developer
- [X] T030 [US1] Create `fe/app/ui/import/import-provider.tsx` (`'use client'`) and
  `fe/app/ui/import/import-notice.tsx`.
  - **Context** `useImport()`: `{ stage, open(lookups), close(), toPick(), toReview(fileName, rows), rows, setRows, fileName, lookups, idempotencyKey, notice, setNotice, rowOpen, setRowOpen }`
    (data-model.md "Import session").
  - **Key**: `idempotencyKey` is a new `crypto.randomUUID()` on each `toReview`.
  - **Rendering**: the provider renders `ImportNotice`, then its children, then `ImportDialog`
    when the stage is not `closed`.
  - **`ImportNotice`**: a `role="status"` line styled as in the design's notice (pill, brand
    tint), with a dismiss button labelled "Dismiss". The notice clears on dismiss and on the next
    `open`.

  Make T024 (provider part) green · skill: frontend-developer
- [X] T031 [US1] Wire the provider and button into the page:
  - **`fe/app/spendings/page.tsx`**: wrap the page content in `<ImportProvider>` **outside**
    `<Suspense>`, so state survives a month change (research R-010).
  - **`fe/app/ui/import/import-csv-button.tsx`** (`'use client'`): a full-width outlined brand
    button with the upload icon, min height 44 px, as in the design artboard "Import CSV window".
    It calls `open(lookups)` and is disabled when `rowOpen`.
  - **`fe/app/ui/spendings/month-section.tsx`**: render `ImportCsvButton` first in the
    `aside aria-label="Month summary"`, above `MonthTotalCard`, passing the same lookups that
    `SpendingsTable` gets.

  Make T024 green · skill: frontend-developer
- [X] T032 [US1] Create `fe/app/ui/import/pick-stage.tsx` (research R-012):
  - **Content**: the design's layout and copy.
  - **Drop area**: the whole area is a drop target; `onDragOver` calls `preventDefault` and sets
    the highlight.
  - **File input**: visually covers "Choose file", has the label "Choose a CSV file" and
    `accept=".csv,text/csv"`.
  - **Handling a file**: take the first file → `checkFile` → `await file.text()` (catch →
    `unreadable`) → `readImport(text, lookups)` → error message or `toReview(file.name, rows)`.
  - **Errors** go in a `role="alert"` paragraph.
  - **Footer**: Cancel and the header close button (`aria-label="Close import"`) call `close()`.
    Leave a slot for the sample link (T045).

  Make T025 green · skill: frontend-developer
- [X] T033 [US1] Create `fe/app/ui/import/import-row.tsx` (`'use client'`), a first version that
  edits values. It is a table row (`role="row"`) with:
  - a `DatePicker` (`fe/app/ui/spendings/date-picker.tsx`) for the date;
  - `Combobox` (`fe/app/ui/spendings/combobox.tsx`) for merchant, category and account, with the
    same lookups as the entry row;
  - an amount `input` (`inputmode="decimal"`);
  - accessible names `Merchant, row N` etc., as in the design.

  Every change calls `onChange(key, patch)`. Highlighting, keyboard rules and remove come in US2
  · skill: frontend-developer
- [X] T034 [US1] Create `fe/app/ui/import/review-stage.tsx`:
  - **Header**: the title "Review import", `{fileName} · {rowCount}`, and a close button labelled
    "Cancel import".
  - **Badge**: "All rows look good" or `needAttention(n)`.
  - **Hint line**: "Highlighted fields need your attention. Remove a row to leave it out."
  - **Table**: `role="table" aria-label="Rows to import"`, with a min width of 840 px inside a
    scroll box (FR-025).
  - **Footer**:
    - `{plural spendings} · total {formatCents(sum)}`;
    - Cancel;
    - the add button, using `addLabel(n)` and `disabled` unless every row is valid and nothing is
      pending.
  - **Add**: `startTransition` → `importSpendings` → on `ok`:
    1. `close()`;
    2. `setNotice(added(count, fileName))`;
    3. `router.push('/spendings?month=' + month)`.

  Make T026 green · skill: frontend-developer
- [X] T035 [US1] Create `fe/app/ui/import/import-dialog.tsx` (`'use client'`):
  - **Element**: a native `<dialog>` opened with `showModal()` in an effect, labelled by the
    stage's heading (`imp-h` / `rev-h`), sized and styled as the design's modal.
  - **Stages**: renders `PickStage` or `ReviewStage` by stage.
  - **Closing**: the `close` event → `close()`.

  Run T027 and make it green; quickstart §3 steps 1, 2, 4 (clean file) and 8 pass by hand
  · skill: frontend-developer

**Checkpoint**: US1 works end to end. MVP: a clean file can be imported.

---

## Phase 4: User Story 2 — Fix and complete rows before adding (Priority: P1)

**Goal**: Invalid fields are highlighted and counted. The user fixes them in place with the
Spendings table's calendar, autofill and keys, removes unwanted rows, and "Add spendings" stays
greyed out until every row is valid. Server refusals land on the right row and field.

**Independent Test**: spec.md US2. Import a file with one row each missing a category, an
account, a date and an amount. Check "4 rows need attention", the four highlights and the greyed
button. Fix them one by one and watch the badge, total and button update. Add, and check the
server holds the edited values.

### Tests (write first, see them fail)

- [X] T036 [P] [US2] Create `fe/app/ui/spendings/row-keys.test.ts`. Move the arrow-step and
  next-field cases out of `entry-row.test.tsx` into it, for `arrowStep(event)`,
  `nextField(field, direction)` and `merchantFillIn(draft, merchant)`:
  - `merchantFillIn` fills an empty category and account from `lastCategory`/`lastAccount`.
  - It never replaces a non-empty value (003 FR-058).

  `entry-row.test.tsx` must still pass unchanged in behaviour · skill: frontend-developer
- [X] T037 [P] [US2] Create `fe/app/ui/import/import-row.test.tsx` (RTL + user-event):
  - **Highlighting**:
    - An empty field shows the placeholder "Required" and has `aria-invalid="true"` and
      `aria-describedby` naming its message.
    - An empty date button reads "Pick a date".
    - A future date is invalid with `Pick a date that is today or earlier.`
  - **Fixing**: typing a valid value clears `aria-invalid` immediately.
  - **Autofill**:
    - Typing `g` in category autofills the most used `G…` category, as in the entry row.
    - Accepting a past merchant fills an empty category and account, but not filled ones.
  - **Keys**:
    - Tab and Shift+Tab, and Left/Right at the text edges, move through date, merchant,
      category, account, amount.
    - Enter in a field moves to the next field.
    - Enter in amount moves focus to the next row's date button.
    - Escape with a list open closes only the list.
  - **Calendar**: opens at the row's date, or at the current month when the date is empty; days
    after `today` are disabled.
  - **Popups shown in full** (FR-025): render the row in the review's scroll box inside an open
    `<dialog>`, scrolled so the row is the last visible one. Open its calendar, then its category
    list. Each popup's bounding box lies inside the viewport and is not cut off by the scroll
    box (assert it is positioned with feature 003's `use-floating.ts`, `position: fixed`, not
    inside the overflow container's clipping).
  - **Remove**: the button labelled `Remove row N` calls `onRemove(key)` without confirmation.
  - **Server error**: a `serverError` for a field highlights it with that message until the
    field is edited.

  · skill: frontend-developer
- [X] T038 [P] [US2] Extend `fe/app/ui/import/review-stage.test.tsx`:
  - **Badge and button**: with 4 invalid rows of 10, the badge reads "4 rows need attention",
    the button reads "Add spendings", has `disabled`, and does nothing on click or Enter. The
    total sums only valid amounts.
  - **Fixing rows**: fixing each row decrements the badge. After the last fix, "All rows look
    good" and "Add 10 spendings" appear, enabled.
  - **Removing rows**: removing a row updates the row count, total and label. Removing all rows
    shows "No rows left to import." and a disabled button.
  - **Server refusal**: `importSpendings` returning `rowErrors` highlights those rows' fields
    and keeps all rows.
  - **When the key changes** (R-013):
    - After a **422 with `rowErrors`** (the server refused, so nothing was saved), the first edit
      makes a new idempotency key.
    - After **"unreachable" or a 5xx** (the server may have saved), the key is kept even if rows
      are edited.
    - When the retry then gets `reason: idempotency-key-reused`, the review shows
      `MESSAGES.alreadyImported`, closes, and refreshes the Spendings screen. It does not add
      again.

  · skill: frontend-developer
- [X] T039 [P] [US2] Add to `fe/e2e/import-csv.spec.ts` and `fe/e2e/fixtures/gaps.csv` (one row
  each missing category, account, date and amount). The test:
  1. Expect "4 rows need attention" and a disabled "Add spendings".
  2. Fill each gap using the keyboard and autofill, with no mouse in the fields.
  3. Remove one row.
  4. Add, and expect the edited values on the Spendings screen.

  · skill: frontend-developer

### Implementation

- [X] T040 [US2] Create `fe/app/ui/spendings/row-keys.ts` with `arrowStep`, `nextField` and
  `merchantFillIn`, moved out of `fe/app/ui/spendings/entry-row.tsx`. Update `entry-row.tsx` to
  import them, with no behaviour change. Make T036 and `entry-row.test.tsx` green
  · skill: frontend-developer
- [X] T041 [US2] Complete `fe/app/ui/import/import-row.tsx`:
  - **Problems**: `problems = { ...draftProblems(draft, today), ...serverErrors }`, where a
    server error is dropped when its field is edited.
  - **Invalid fields**: the design's warning border and background on each invalid field,
    `aria-invalid`, and the message in a visually hidden element referenced by
    `aria-describedby`. The message is also shown as a small line under the row while that row
    has focus (R-008).
  - **Keyboard** (via `row-keys.ts`): Enter moves to the next field, and from amount to the next
    row's date (use a `data-import-row` index and focus the next row's date button). Escape
    inside an open combobox or calendar closes it and calls `stopPropagation`.
  - **Remove button**: the design's icon, `aria-label="Remove row N"`, `title="Remove this row"`.
  - **Popups**: the `DatePicker` and `Combobox` popups must use `use-floating.ts` exactly as in
    the Spendings table, so the review's scroll box and the dialog never clip them (FR-025). If
    the top-layer `<dialog>` breaks fixed positioning (for example because of a `transform` on
    the dialog), remove the transform; don't clip.

  Make T037 green · skill: frontend-developer
- [X] T042 [US2] Complete `fe/app/ui/import/review-stage.tsx`:
  - **Problem counts**: count rows with problems for the badge; `canAdd` comes from data-model.md
    "Derived review figures".
  - **Empty table**: "No rows left to import."
  - **Server errors**: keep them by row key from `rowErrors`.
  - **New key**: call `regenerateKey()` on the first edit after a failed add **only when the
    failure was a 422 with `rowErrors`**; keep the key after "unreachable" or a 5xx. On
    `{ status: 'already-imported' }` from the action, close the dialog, show
    `MESSAGES.alreadyImported` as the notice, and `router.refresh()`.
  - **Remove**: `setRows(rows.filter(…))`.

  Make T038 and T039 green · skill: frontend-developer

**Checkpoint**: US1 and US2 work. Imperfect files can be fixed and imported.

---

## Phase 5: User Story 3 — Download the sample file (Priority: P2)

**Goal**: A "Download sample CSV" link in the Import CSV window saves `sample-spendings.csv`,
which shows the format and imports cleanly into the review with its gaps highlighted.

**Independent Test**: spec.md US3. Click "Download sample CSV"; a 10-row file is saved and the
window stays open. Importing that file shows the same 10 rows.

### Tests (write first, see them fail)

- [X] T043 [P] [US3] Extend `fe/app/lib/csv-import.test.ts`. Read
  `fe/public/sample-spendings.csv` from disk with `fs`, run `readImport`, and assert all 10 rows
  exactly as research R-011 implies:
  - `Dr. Patel Dental` has category `''`.
  - `Birch Yoga Studio` has account `''`.
  - `Thrift & Co.` has amount `''`.
  - `Ride share` has amount `18.90`.
  - `Lakeshore Fuel, Hwy 7` keeps its comma.
  - `Hardware Depot` has date `2026-10-03` and amount `1041.18`.
  - `Harbour Coffee` has date `2026-10-27`.

  · skill: frontend-developer
- [X] T044 [P] [US3] Extend `fe/app/ui/import/pick-stage.test.tsx`: there is a link "Download
  sample CSV" with `href="/sample-spendings.csv"` and the `download` attribute, and clicking it
  does not close the dialog or change the stage. Extend `fe/e2e/import-csv.spec.ts`:
  - Clicking the link raises a Playwright `download` event named `sample-spendings.csv`.
  - The saved file starts with `Date,Merchant,Category,Account,Amount`.
  - Importing that saved file shows `sample-spendings.csv · 10 rows`.

  · skill: frontend-developer

### Implementation

- [X] T045 [US3] Create `fe/public/sample-spendings.csv` with exactly the 11 lines in research
  R-011 (UTF-8, `\n` line endings, trailing newline). In `fe/app/ui/import/pick-stage.tsx`, put
  the link in the footer where the design had "Try a sample file", styled as the design's text
  button. Check that `fe/proxy.ts` does not gate `/sample-spendings.csv` (static assets are
  excluded by its matcher; add a test case to `fe/proxy.test.ts` if not already covered). Make
  T043 and T044 green · skill: frontend-developer

**Checkpoint**: US3 works independently of the review's editing.

---

## Phase 6: User Story 4 — Cancel an import (Priority: P2)

**Goal**: Cancelling from either window saves nothing. The review goes back to the pick window;
the pick window goes back to the Spendings screen.

**Independent Test**: spec.md US4. Import a file, edit rows, and cancel; no spending, category or
account is created. Reopening the file shows its original contents.

### Tests (write first, see them fail)

- [X] T046 [P] [US4] Create `fe/app/ui/import/import-dialog.test.tsx` (RTL, with
  `HTMLDialogElement.prototype.showModal`/`close` stubbed as in the settings dialog tests):
  - **Pick stage**: Cancel, "Close import" and Escape (a `cancel` event) all call `close()`.
    The page behind is not focusable (the dialog is modal).
  - **Review stage**:
    - Cancel and "Cancel import" call `toPick()`; the rows and edits are gone.
    - Escape with no popup open calls `preventDefault` on `cancel` and calls `toPick()`.
    - Escape with a calendar or combobox open closes only the popup.
  - **Reopening** the same file after cancelling shows the file's values, not the earlier edits.
  - **No API call** is made in any of these cases.

  · skill: frontend-developer
- [X] T047 [P] [US4] Add to `fe/e2e/import-csv.spec.ts`. Import `gaps.csv`, type into two rows,
  press Escape (back to the pick window), then click Cancel (back to the Spendings screen).
  Assert:
  - the month totals are unchanged;
  - Settings shows no new category or account;
  - re-importing shows the original gaps.

  · skill: frontend-developer

### Implementation

- [X] T048 [US4] In `fe/app/ui/import/import-dialog.tsx`, handle the dialog's `cancel` event:
  - In the review stage, `preventDefault()` and `toPick()`; popups already stop the Escape
    themselves (T041).
  - In the pick stage, let it close and call `close()`.

  Wire the review's Cancel and close buttons to `toPick()`, and make `toPick()` clear the rows,
  the file name, the server errors and the key. Make T046 and T047 green
  · skill: frontend-developer

**Checkpoint**: All four stories work independently.

---

## Phase 7: Polish & Cross-Cutting Concerns

**Purpose**: The edge cases spanning stories, performance and accessibility proof, docs, gates
and release.

- [X] T049 [P] Disable the Import CSV button while a row is open (spec Edge Case "Open entry
  row"):
  - In `fe/app/ui/spendings/spendings-table.tsx`, call `setRowOpen(true/false)` from
    `useImport()` whenever the add or edit row opens or closes.
  - In `fe/app/ui/spendings/spendings-table.test.tsx`, wrap the table in the provider and assert
    the button is disabled while a row is open.
  - Add an e2e step in `fe/e2e/import-csv.spec.ts`: click +, and the button is disabled;
    Escape, and it is enabled.

  · skill: frontend-developer
- [X] T050 [P] Create `fe/e2e/fixtures/make-csv.mjs`, a Node script that prints N valid rows to
  stdout (`node e2e/fixtures/make-csv.mjs 1000`). Use day-first dates in the last 90 days,
  rotating 5 merchants, existing default categories, account `Visa`, amounts `1.00`–`99.99`, and
  the e2e marker. Add these e2e steps to `fe/e2e/import-csv.spec.ts`:
  - Refusals with the fixtures `not-csv.png`, `empty.csv` and `header-only.csv`, plus a
    generated 1,001-row file and a generated file just over 1 MB. Each shows its spec message
    and no review opens.
  - **SC-005**: a generated 1,000-row file shows `1,000 rows` in the review within 3 s of
    `setInputFiles`, and typing in a field updates the badge within 200 ms. Do not add these
    rows; cancel.

  · skill: frontend-developer
- [X] T051 [P] Write `…/test/controller/SpendingImportPerformanceIT.kt`. Import 1,000 valid rows
  across 20 category names and 3 account names (half of them new). Assert:
  - `201` in **< 2 s** (median of 3 runs, each with a new key, after a warm-up);
  - exactly 1,000 new spendings per run;
  - each new name created once.

  · skill: backend-developer
- [X] T052 [P] Extend `fe/e2e/a11y.spec.ts`: axe scans of the pick stage, the pick stage with an
  error shown, the review stage with highlighted rows, and the review with a calendar and a
  combobox list open. There must be no `serious` or `critical` violations (FR-052). Also check
  by keyboard alone that Tab never leaves the dialog. With 30 rows loaded, scroll to the last
  row, open its calendar and its category list, and assert each popup is fully inside the
  viewport (FR-025) · skill: frontend-developer
- [X] T053 [P] Update the READMEs:
  - `hl-budget-backend/README.md`: the `spending-imports` endpoints, all-or-nothing behaviour,
    idempotency and the V4 table.
  - `hl-budget-frontend/README.md`: the import dialog, the sample file, `make-csv.mjs` and the
    new e2e spec.

  · skill: backend-developer, frontend-developer
- [ ] T054 _(partly done 2026-10-29: `./gradlew check`, `pnpm typecheck && pnpm lint && pnpm test` (333 tests) and `pnpm build` are green, and the image build was checked for `public/sample-spendings.csv`. Still to run: `pnpm e2e` (needs the AUTH0_*/API env, a running backend and `E2E_USERNAME`/`E2E_PASSWORD`) and quickstart §2–§3 by hand.)_ Run the full gates:
  - **Backend**: `./gradlew check`.
  - **Frontend**: `pnpm typecheck && pnpm lint && pnpm test && pnpm build && pnpm e2e`.

  Then run quickstart §2 and §3 by hand, including step 11 (backend stopped, then retried: added
  once) · skill: backend-developer, frontend-developer
- [ ] T055 Operator step (ask the user first). For each service, open the PR, merge it after
  review, let the release workflow open the hl-helm image PR, and merge that. **Backend first**:
  Flyway applies V4, and the new endpoints are inert for the running frontend. Then the frontend
  · skill: none
- [ ] T056 Verify both rollouts with `homelab:hl-deploy-status`: Argo CD Synced/Healthy, pods
  Ready, backend `/readyz` 200 after V4. Then run the live check from quickstart §4.3 with the
  sample file and delete the imported spendings afterwards. Record the result in the **Status**
  line of `specs/004-csv-import/spec.md` · skill: homelab:hl-deploy-status
- [ ] T057 Update the Features table in the spec repo's `README.md` with feature 004, in a small
  docs PR (`docs: list feature 004 in the README`) · skill: none

---

## Implementation notes (2026-10-29)

- **Popups inside the import window**: the combobox and date picker portalled to `document.body`,
  which a modal `<dialog>` hides behind its top layer. They now portal into the enclosing dialog
  through a `FloatingRoot` context (`app/ui/spendings/use-floating.ts`); outside a dialog nothing
  changes (C1 from the analysis).
- **Sample file in the image**: the standalone Dockerfile did not copy `public/`, so the sample
  would have 404'd after deploy. `COPY --from=build /app/public ./public` was added.
- **Date problems are described, not `aria-invalid`**: a `<button>` cannot carry `aria-invalid`
  (jsx-a11y), so an invalid date gets the error border plus `aria-describedby` with its message.
- **Server refusals**: the footer message shows only failures without row errors (for example
  "Couldn't reach the server"); refused rows are counted by the badge and explained per field.
- **`SpendingImport` fields are explicitly required** in the OpenAPI document
  (`requiredMode = REQUIRED`), so the generated frontend type has no optional fields.

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: no dependencies.
- **Foundational (Phase 2)**: after T001. Blocks every frontend story task. The backend tasks
  T010–T022 do not depend on it and can run in parallel with it.
- **US1 (Phase 3)**: the frontend part needs Phase 2, and T022 (regenerated types) for
  T028–T029.
- **US2 (Phase 4)**: needs US1's `ImportRow` and `ReviewStage` (T033, T034).
- **US3 (Phase 5)**: needs US1's `PickStage` (T032). T043 needs only Phase 2.
- **US4 (Phase 6)**: needs US1's dialog (T035). T048 builds on T041's popup Escape handling.
- **Polish (Phase 7)**: after the stories it touches. T051 needs only T021.

### User Story Dependencies

- **US1 (P1)**: Phase 2 plus the backend. It is the MVP.
- **US2 (P1)**: builds on US1's review table.
- **US3 (P2)**: independent of US2 and US4. Needs only the pick window.
- **US4 (P2)**: independent of US2 and US3. Needs only the dialog; its Escape test with an open
  popup uses US1's `DatePicker`/`Combobox`.

### Within Each User Story

- Tests first; see them fail.
- Backend: migration → persistence → rules and resolver → service → controller → contract
  export.
- Frontend: definitions/mocks → action → provider → stages → dialog.
- Commit after each task or logical group.

### Parallel Opportunities

- **Phase 2**: the tests T002–T005 together; then T006, T008 and T009 together, with T007 after
  T006.
- **Backend track**: T010–T013 together, then T015, T016 and T019 together alongside T014.
- **US1 frontend tests**: T023–T027 together.
- **US2–US4**: their test tasks are each [P]. Once US1 is done, US3 and US4 can be built in
  parallel with US2.
- **Polish**: T049–T053 together.

---

## Parallel Example: backend track alongside Phase 2

```text
Developer A (backend): T010, T011, T012, T013 → T014 → T015, T016, T019 → T017 → T018 → T020 → T021 → T022
Developer B (frontend): T002, T003, T004, T005 → T006, T008, T009 → T007
Both meet at T028 (needs T022's regenerated types).
```

## Parallel Example: User Story 1 frontend

```text
Task: "T023 importSpendings action tests in fe/app/lib/actions.test.ts"
Task: "T024 provider and button tests in fe/app/ui/import/"
Task: "T025 pick stage tests in fe/app/ui/import/pick-stage.test.tsx"
Task: "T026 review stage tests in fe/app/ui/import/review-stage.test.tsx"
Task: "T027 e2e US1 in fe/e2e/import-csv.spec.ts"
```

---

## Implementation Strategy

### MVP First (User Story 1)

1. Phase 1, then Phase 2 and the backend track in parallel.
2. Phase 3 (US1): a clean file imports all-or-nothing and lands on the right month.
3. **Stop and validate**: quickstart §2 (backend) and §3 steps 1, 2, 4 and 8 with a clean file.
4. The backend can be released at this point; it is inert for the current frontend.

### Incremental Delivery

1. US1 → a clean import works.
2. US2 → real exports with gaps can be fixed and imported. This is the minimum worth releasing
   to the homelab, since bank files are rarely perfect.
3. US3 → sample download.
4. US4 → cancel paths.
5. Polish → edge cases, performance, axe, docs, release (T055–T057).
