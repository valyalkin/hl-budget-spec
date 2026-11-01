---

description: "Task list for 007-fix-pie-csv-categories"
---

# Tasks: Fixed category colours, a working pie chart, and categories and accounts managed only in Settings

**Input**: Design documents from `/specs/007-fix-pie-csv-categories/`

**Prerequisites**: plan.md, spec.md, research.md, data-model.md, contracts/openapi-delta.yaml,
quickstart.md

**Tests**: REQUIRED. Constitution Principle IV: tests ship with the code, and every changed
user-visible flow keeps a Playwright spec. Within each phase, write the tests first and see them
fail before writing the code that satisfies them.

**Organization**: Tasks are grouped by user story (spec.md US1–US4). The backend work for a story
lives in that story's phase. US1 needs backend colours; US2 is frontend only; US3 and US4 need
the backend name rule.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Can run in parallel (different files, no dependency on an unfinished task).
- **[Story]**: The user story the task belongs to (US1 … US4).
- Every task ends with `· skill: <name>`. Load that skill before starting; its definition of done
  is part of the task's done. Where a skill and the constitution disagree, the constitution wins.

## Path Conventions

All paths are relative to the spec repo root `/Users/valyalkin/Documents/Projects/hl-budget-spec/`.

- `hl-budget-backend/`: git worktree of `valyalkin/hl-budget-backend`, on branch
  `007-fix-pie-csv-categories` (T001).
  - Main code: `hl-budget-backend/src/main/kotlin/com/valyalkin/hlbudget/`, written `…/main/`.
  - Tests: `hl-budget-backend/src/test/kotlin/com/valyalkin/hlbudget/`, written `…/test/`.
  - Migrations: `hl-budget-backend/src/main/resources/db/migration/`, written `…/migration/`.
- `hl-budget-frontend/`: git worktree of `valyalkin/hl-budget-frontend`, on branch
  `007-fix-pie-csv-categories` (T001). Written `fe/`.
- **Commits**: inside the service repos, Conventional Commits (`fix:`, `feat:`, `test:`), after
  each task or logical group.
- **Merging**: merging to `main` in a service repo triggers a release. Do it only after the gate
  is green and the user agrees.

Shared rules that several tasks reference:

- **Regenerate types**: run `./gradlew check` in `hl-budget-backend/`; `OpenApiExportIT` rewrites
  `hl-budget-backend/contracts/api.yaml`. Then
  `cp hl-budget-backend/contracts/api.yaml hl-budget-frontend/contracts/api.yaml` and, in `fe/`,
  `pnpm generate:api`. Commit both frontend files.
- **Palette** (research R-003), in this order. Positions 1–17 belong to the defaults in
  `DefaultCategories` order; 18–24 are spare:
  `#1e5b45` Rent, `#c2410c` Groceries, `#1d4ed8` Public Transport, `#a16207` Car,
  `#7e22ce` Household, `#0f766e` Health, `#be185d` Beauty, `#4d7c0f` Clothes,
  `#b91c1c` Education, `#0369a1` Travel, `#57534e` Bills, `#4338ca` Subscriptions,
  `#64748b` Other, `#16a34a` Sport, `#c026d3` Entertainment, `#7c2d12` Pet, `#8a6d00` Taxi,
  then spare `#e11d48`, `#0891b2`, `#a855f7`, `#365314`, `#1e3a8a`, `#701a75`, `#ea580c`.
- **Colour assignment** (R-004): for a new custom category, count the caller's categories per
  palette colour; pick the least-used; ties go to the first in the order 18…24, then 1…17.
- **Name match** (unchanged): `lower(trim(name)) == lower(item.name)`. Frontend uses
  `normaliseName` from `fe/app/lib/csv-import.ts` plus lower-casing.
- **Messages**: every user-facing string comes from `fe/app/lib/messages.ts`, worded exactly as in
  spec.md: "Not one of your categories", "Not one of your accounts", "Add an account in Settings
  first.", "Open Settings".
- **Not yours = not found** (backend): every new query filters on the caller's `userId`.
- **No colour by CSS variable**: no `var(--color-cat-…)` anywhere after this feature (R-001).
  Colours are `#rrggbb` strings set as `fill` / `background` directly.
- **Client components**: no `useMemo`, `useCallback` or `React.memo` (the React Compiler is on).

---

## Phase 1: Setup

**Purpose**: Fresh feature branches with green gates.

- [X] T001 In both `hl-budget-backend/` and `hl-budget-frontend/`:
  1. `git fetch origin` and `git switch -c 007-fix-pie-csv-categories origin/main` (if the
     worktree is on another branch with no uncommitted work, switch it; otherwise ask the user).
  2. Confirm the gates are green before any change: `./gradlew check` in the backend;
     `pnpm install && pnpm typecheck && pnpm lint && pnpm test` in the frontend.

  In the spec repo, create branch `007-fix-pie-csv-categories` from `main` and ask the user
  before committing `specs/007-fix-pie-csv-categories/` · skill: none

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: The shared frontend name check used by US3 and US4, and the new messages. US1 and
US2 do not depend on this phase and may start in parallel with it.

### Tests first (see them fail)

- [X] T002 [P] Extend `fe/app/lib/draft.test.ts` for
  `draftProblems(draft, today, lookups: { categories: readonly { name: string }[]; accounts: readonly { name: string }[] })`:
  - category "Groceriess" with lookups containing "Groceries" →
    `problems.category === MESSAGES.unknownCategory`;
  - category "  groceries " → no category problem;
  - empty category → `MESSAGES.category` (unchanged); 51 characters → `MESSAGES.category`
    (unchanged);
  - account "Visa Gold" with lookups `["Visa"]` → `MESSAGES.unknownAccount`;
  - `draftProblem(draft, today, lookups)` returns the first problem in field order, as today.
  · skill: frontend-developer
- [X] T003 [P] Extend `fe/app/lib/messages.test.ts`: `MESSAGES.unknownCategory` is "Not one of
  your categories", `MESSAGES.unknownAccount` is "Not one of your accounts",
  `MESSAGES.noAccounts` is "Add an account in Settings first.", `MESSAGES.openSettings` is "Open
  Settings". `messageFor` on a 422 whose first violation has `code: "NotFound"` and
  `field: "categoryName"` returns `MESSAGES.unknownCategory` (`accountName` →
  `MESSAGES.unknownAccount`); other 422s are unchanged · skill: frontend-developer

### Implementation

- [X] T004 Add the four messages to `fe/app/lib/messages.ts`. In `messageFor`, before the generic
  422 branch, map a first violation with `code === 'NotFound'` on `categoryName`/`categoryId` to
  `unknownCategory` and on `accountName`/`accountId` to `unknownAccount`. Export a helper
  `violationMessage(violation: { field?: string; code?: string }): string | undefined` that does
  the same mapping and falls back to `fieldMessage(field)`, for reuse by T040 · skill:
  frontend-developer
- [X] T005 Change `fe/app/lib/draft.ts`: add the `DraftLookups` type and a third parameter
  `lookups` to `draftProblems` and `draftProblem`. A non-empty category (≤ 50 characters) with no
  match in `lookups.categories` gets `MESSAGES.unknownCategory`; the same for accounts with
  `MESSAGES.unknownAccount`. Add `matchItem(name, items)` that returns the matched item (or
  `undefined`), using the shared name-match rule. Update every caller to pass lookups so the
  typecheck passes: `fe/app/ui/spendings/entry-row.tsx`, `fe/app/ui/import/review-stage.tsx`,
  `fe/app/lib/actions.ts` (both call sites), `fe/app/lib/phone.ts` (three call sites). The callers
  already have the category and account lists in scope or in their props; where a Server Action
  does not (T026), take them as an input field · skill: frontend-developer

**Checkpoint**: `pnpm typecheck && pnpm test` green; nothing on screen changes yet except unknown
names now show the new message.

---

## Phase 3: User Story 1 — Every category has its own colour in the pie (Priority: P1) 🎯 MVP

**Goal**: Each default category has the same fixed colour for every user and month; each new
category gets an unused colour and keeps it; slices, legend and table swatches use it, on desktop
and phone.

**Independent Test**: quickstart §3.1–3.3 and §4.1. A month with spending in 12+ categories shows
every slice coloured and matching its swatch; Groceries keeps its colour across months; a new
category "Coffee" gets `#e11d48`.

### Backend tests (write first, see them fail)

- [X] T006 [P] [US1] Create `…/test/model/CategoryPaletteTest.kt`:
  - `CategoryPalette.COLORS` has 24 entries, all distinct, all matching `^#[0-9a-f]{6}$`, in the
    Palette order above;
  - each has WCAG contrast ≥ 3.0 against `#ffffff` (compute relative luminance in the test);
  - `DefaultCategories.ALL` has 17 entries whose colours are positions 1–17, all distinct, Rent
    `#1e5b45`, Taxi `#8a6d00`;
  - `CategoryPalette.next(counts: Map<String, Int>)`: with the 17 default colours at 1 and spares
    at 0 → `#e11d48`; then successive calls (adding each result to counts) give 18…24, then
    `#e11d48` again (all at 1, tie order starts at 18), then `#0891b2`…; with a spare freed
    (count 0) it is chosen first · skill: backend-developer
- [X] T007 [P] [US1] Create `…/test/repository/CategoryColorMigrationIT.kt` (Testcontainers,
  Flyway `target("4")` then migrate to latest): seed one user with the 17 defaults and 10 custom
  categories with increasing `created_at`, a second user with defaults only. After V5: defaults
  have their palette colours by name; the first user's customs, in creation order, have positions
  18, 19, 20, 21, 22, 23, 24, 18, 19, 20; `color` is NOT NULL; inserting `'#ZZZZZZ'` or
  `'#ABCDEF'` (upper case) violates the check · skill: backend-developer
- [X] T008 [P] [US1] Update `…/test/controller/DefaultCategoriesIT.kt` and
  `…/test/controller/CategoryControllerIT.kt`:
  - a new user's 17 categories each have `color` equal to its palette colour; a second user gets
    the same colours;
  - `POST /api/v1/categories {"name":"Coffee"}` → `201`, `color` `#e11d48`; a second new
    category → `#0891b2`; renaming Coffee keeps `#e11d48`; deleting Coffee and creating "Tea" →
    `#e11d48`;
  - `GET /api/v1/categories/{id}` and the list include `color` · skill: backend-developer
- [X] T009 [P] [US1] Update `…/test/controller/TotalsIT.kt`: every `GET /api/v1/category-totals`
  item has `color` equal to its category's `color` · skill: backend-developer

### Backend implementation

- [X] T010 [US1] Create `…/main/model/CategoryPalette.kt`: `object CategoryPalette` with
  `COLORS: List<String>` (the Palette, 24 values), `DEFAULTS = COLORS.take(17)`,
  `SPARE = COLORS.drop(17)`, and `fun next(counts: Map<String, Int>): String` (least-used; ties
  by order `SPARE + DEFAULTS`). Change `…/main/model/DefaultCategories.kt`: replace `NAMES` with
  `data class DefaultCategory(val name: String, val color: String)` and
  `ALL: List<DefaultCategory>` zipping the 17 names with `CategoryPalette.DEFAULTS`; keep a
  derived `NAMES` only if other code still needs it · skill: backend-developer
- [X] T011 [US1] Create `…/migration/V5__add_category_colors.sql` (research R-005):
  1. `ALTER TABLE categories ADD COLUMN color varchar(7);`
  2. Update defaults from a `VALUES (name, color)` list of the 17 pairs, joined on
     `is_default = true AND name = v.name`.
  3. Update custom categories: in a CTE, `row_number() OVER (PARTITION BY user_id ORDER BY
     created_at, id) AS rn` where `is_default = false`; colour = the palette entry at 1-based
     position `17 + rn` when `rn <= 7`, else the entry at 0-based index `(rn - 8) % 24` of the
     order `[18..24, 1..17]` (express that 24-entry order as a `VALUES (idx, color)` list).
     Check: rn 1 → 18, rn 7 → 24, rn 8 → 18, rn 14 → 24, rn 15 → 1.
  4. `ALTER TABLE categories ALTER COLUMN color SET NOT NULL;` and
     `ADD CONSTRAINT ck_categories_color CHECK (color ~ '^#[0-9a-f]{6}$');`
  Comment the file with a pointer to research R-004/R-005 · skill: backend-developer
- [X] T012 [US1] Add `color` to `…/main/repository/category/CategoryEntity.kt`
  (`@Column(name = "color", nullable = false, updatable = false, length = 7)`), to
  `…/main/model/Category.kt`, and through `…/main/repository/category/CategoryMappings.kt` and the
  `findWithCount` projection in `…/main/repository/category/CategoryRepository.kt`. Add
  `colorCounts(userId): Map<String, Int>` to the repository (`SELECT color, count(*) … WHERE
  user_id = :userId GROUP BY color`). Change
  `…/main/repository/CurrentUserRepository.kt#insertDefaultCategories(userId, categories:
  List<DefaultCategory>)` to insert `color` too, and pass `DefaultCategories.ALL` from
  `…/main/service/CurrentUserService.kt` · skill: backend-developer
- [X] T013 [US1] In `…/main/service/CategoryService.kt#create`, set
  `color = CategoryPalette.next(categories.colorCounts(userId))` on the new entity and return it
  in the model. Rename does not touch `color` · skill: backend-developer
- [X] T014 [US1] API fields:
  - `…/main/dto/category/CategoryResponse.kt`: add `color: String`, required, read-only,
    `pattern = "^#[0-9a-f]{6}$"`, description and example `#1e5b45` as in
    contracts/openapi-delta.yaml; map it in `…/main/dto/category/CategoryMappings.kt`.
  - `…/main/repository/spending/TotalsQueries.kt#byCategory`: select `c.color` into
    `CategoryTotalRow.categoryColor`; `…/main/model/Totals.kt#CategoryTotal` gains `color`;
    `…/main/service/TotalsService.kt` passes it; `…/main/dto/totals/TotalsResponses.kt
    #CategoryTotalResponse` gains required `color` with description and example `#c2410c`.
  · skill: backend-developer
- [X] T015 [US1] Run `./gradlew check` (T006–T009 now pass, Kover floors hold), then **Regenerate
  types** · skill: backend-developer

### Frontend tests (write first, see them fail)

- [X] T016 [P] [US1] Update `fe/app/ui/spendings/by-category-card.test.tsx`: given items with
  `color`, each pie slice's `fill` and each legend swatch's background equal that item's colour;
  no element has a `fill` or style containing `var(--color-cat`; Table view shows one swatch per
  row with the item's colour · skill: frontend-developer
- [X] T017 [P] [US1] Update `fe/app/lib/phone.test.ts` (`topCategories`) and
  `fe/app/ui/phone/category-breakdown.test.tsx`: each top-6 slice carries its item's `color`; the
  "Everything else" slice is `#8a908b`; donut slices use those colours · skill:
  frontend-developer
- [X] T018 [P] [US1] Update `fe/e2e/by-category.spec.ts` and the phone by-category spec under
  `fe/e2e/phone/`: with spendings in 12 categories, every slice has a computed `fill` that is a
  real colour (not `none`, not black, not transparent) and equals its legend swatch; Groceries has
  the same fill in two different months · skill: frontend-developer

### Frontend implementation

- [X] T019 [US1] Add `color: string` to `CategoryItem` in `fe/app/ui/spendings/category-table.tsx`
  and fill it from `item.color` where totals are mapped: `fe/app/ui/spendings/month-section.tsx`
  and `fe/app/ui/phone/phone-section.tsx` (and the type in `fe/app/ui/phone/phone-data.ts`).
  Update test fixtures that build `CategoryItem`s · skill: frontend-developer
- [X] T020 [US1] Desktop colours: in `fe/app/ui/spendings/category-pie.tsx` use `item.color` for
  slice `fill` and legend swatch; delete `fe/app/ui/spendings/category-colors.ts`; in
  `fe/app/ui/spendings/category-table.tsx` add a 12px `aria-hidden` square swatch in
  `item.color` before each name (R-013) · skill: frontend-developer
- [X] T021 [US1] Phone colours: in `fe/app/lib/phone.ts#topCategories` accept
  `items: { id; name; cents; color }[]`, use `item.color` for the top slices, and replace
  `REST_COLOR` with the literal `#8a908b`; remove the `CATEGORY_COLORS` array there · skill:
  frontend-developer
- [X] T022 [US1] Remove `--color-cat-1` … `--color-cat-12` and their comment from
  `fe/app/globals.css`; `grep -rn "color-cat" fe/app fe/e2e` must return nothing. Run
  `pnpm typecheck && pnpm lint && pnpm test` · skill: frontend-developer

**Checkpoint**: US1 works on its own with the backend from T015 running locally.

---

## Phase 4: User Story 2 — Hovering a slice names the category (Priority: P1)

**Goal**: Hover, keyboard focus or tap on a slice shows its name, amount and share in the donut's
centre straight away, with the slice emphasised (research R-012).

**Independent Test**: quickstart §4.2.

### Tests (write first, see them fail)

- [X] T023 [P] [US2] Create `fe/app/ui/charts/donut-chart.test.tsx` for
  `DonutChart({ slices, totalText, summary })` with
  `slices: { key: string; label: string; amountText: string; shareText: string; color: string }[]`:
  - idle: centre shows "Total" and `totalText`; the `<svg>` has `role="group"` and
    `aria-label={summary}`; no `<title>` elements;
  - `pointerenter` on slice 2 → centre shows its `label` and `"{amountText} · {shareText}"` in a
    `role="status"` element; other slices have opacity 0.35; slice 2 has the outline stroke;
  - `pointerenter` on slice 3 → label switches; `pointerleave` on the svg → idle;
  - each slice has `tabIndex=0`, `role="img"`, `aria-label="{label}: {amountText}, {shareText}"`;
    `focus` → same label; `blur` to outside → idle;
  - touch: `pointerdown` with `pointerType: 'touch'` on a slice → label; the same again → idle;
    `pointerdown` on `document.body` → idle;
  - one slice → a full ring, still focusable and labelled · skill: frontend-developer
- [X] T024 [P] [US2] Extend `fe/e2e/by-category.spec.ts` and the phone by-category spec: hover
  (desktop) / tap (phone) a slice → its category name is visible within 500 ms; Tab reaches the
  slices and shows the label; the axe scan of the card in Pie view has no violations · skill:
  frontend-developer

### Implementation

- [X] T025 [US2] Create `fe/app/ui/charts/donut-chart.tsx` (`'use client'`), built on
  `donutArcs` from `fe/app/lib/utils.ts`, meeting T023:
  - state `active: number | null`; a document `pointerdown` listener is registered only while
    `active !== null` and touch-opened;
  - the centre label is an absolutely positioned `<div>` over the svg centre with the
    name wrapping, clamped to the inner circle width;
  - the active slice gets `stroke` `var(--color-ink)` width 2; the others `opacity: 0.35`; the
    slice gap stays (white 1.5 stroke when idle).

  Then use it in `fe/app/ui/spendings/category-pie.tsx` (keep the legend there) and in
  `fe/app/ui/phone/category-breakdown.tsx` (replace its inline svg; keep its legend list), mapping
  items with `formatCents` and `sharePercent` · skill: frontend-developer

**Checkpoint**: US1 + US2 make the chart fully usable. This is the smallest releasable
improvement for the pie bug.

---

## Phase 5: User Story 3 — Spendings use only existing categories and accounts (Priority: P1)

**Goal**: Adding or editing a spending (desktop entry row, phone Add / Edit) accepts only existing
categories and accounts; nothing is created; a user with no accounts is sent to Settings.

**Independent Test**: quickstart §3.4–3.5 and §4.3–4.4.

### Backend tests (write first, see them fail)

- [X] T026 [P] [US3] Update `…/test/controller/SpendingByNameIT.kt`:
  - `POST /api/v1/spendings` with `categoryName: "Groceriess"` → `422`, `type`
    `urn:problem:hl-budget:unprocessable-entity`, one violation `{pointer: "#/categoryName",
    field: "categoryName", code: "NotFound", message: "is not one of your categories"}`; the
    category count is unchanged;
  - `accountName: "Visa Gold"` (absent) → `422` on `#/accountName`, "is not one of your
    accounts"; both unknown → both violations at once; account count unchanged;
  - `categoryName: "  groceries "` → `201`, category "Groceries";
  - `PUT /api/v1/spendings/{id}` with an unknown name → `422`, spending unchanged;
  - remove the old "creates a missing category/account" cases · skill: backend-developer

### Backend implementation

- [X] T027 [US3] In `…/main/service/NamedItemResolver.kt`, make `byName` a lookup: find the
  caller's category/account with `existsByUserIdAndNameIgnoreCase`-style lookup on the trimmed
  name (add `findByUserIdAndNameIgnoreCase` to `CategoryRepository` and `AccountRepository` if
  missing). Collect misses as `FieldViolation("#/categoryName", "categoryName", "NotFound", "is
  not one of your categories")` / the account equivalent, together with earlier violations, and
  throw one `ValidationException` before anything is written. Update the class KDoc. Update the
  `categoryName`/`accountName` `@Schema` descriptions in
  `…/main/dto/spending/SpendingRequest.kt` to the text in contracts/openapi-delta.yaml, and add
  the `unknownCategoryName` 422 example to the spending create/replace `@ApiResponse` in
  `…/main/controller/SpendingController.kt` · skill: backend-developer

### Frontend tests (write first, see them fail)

- [X] T028 [P] [US3] Update `fe/app/ui/spendings/combobox.test.tsx`: no "Create “x”" option ever
  appears (remove `createHint` cases); and `fe/app/ui/phone/suggest-input.test.tsx`: no create
  hint · skill: frontend-developer
- [X] T029 [P] [US3] Update `fe/app/ui/spendings/entry-row.test.tsx` and
  `fe/app/ui/phone/spending-form.test.tsx`:
  - category "Groceriess" → field `aria-invalid="true"` with "Not one of your categories"; Enter /
    Save does not call the save action;
  - "groceries" → shown as "Groceries" on blur, saves with `categoryId` of Groceries;
  - account "Visa Gold" → "Not one of your accounts";
  - with `accounts=[]`: the account field is replaced by "Add an account in Settings first." and an
    "Open Settings" control that opens Settings on Accounts; save is disabled · skill:
  frontend-developer
- [X] T030 [P] [US3] Update `fe/app/lib/actions.test.ts` (spending save): a value whose category
  matches an item sends `categoryId`, never `categoryName`; a server `422` with a `NotFound`
  violation on `categoryName` returns `{ status: 'error', field: 'category', message: 'Not one of
  your categories' }` · skill: frontend-developer
- [X] T031 [P] [US3] Update `fe/e2e/add-spending.spec.ts`, `fe/e2e/keyboard-autofill.spec.ts`,
  `fe/e2e/settings.spec.ts` and the phone add/edit spec under `fe/e2e/phone/`: replace any step
  that relied on typing a new category or account to create it (create it in Settings first);
  add a case typing an unknown category → message shown, nothing saved, Settings counts unchanged
  · skill: frontend-developer

### Frontend implementation

- [X] T032 [US3] Remove `createHint` and `showCreate` from `fe/app/ui/spendings/combobox.tsx` and
  every `createHint` prop at its call sites; remove any create hint from
  `fe/app/ui/phone/suggest-input.tsx` · skill: frontend-developer
- [X] T033 [US3] In `fe/app/ui/spendings/entry-row.tsx` and `fe/app/ui/phone/spending-form.tsx`:
  pass `{ categories, accounts }` lookups to `draftProblem(s)`; on blur of a category/account
  field, replace the text with the matched item's exact name (`matchItem`); when `accounts` is
  empty, render `MESSAGES.noAccounts` and an "Open Settings" button in place of the account field
  (desktop: open the Settings dialog on Accounts; phone: open phone Settings on the Accounts tab,
  using the existing open functions) and disable save · skill: frontend-developer
- [X] T034 [US3] In `fe/app/lib/actions.ts` (spending save, around the `SpendingRequest` body):
  resolve typed names against the submitted lookups with `matchItem` and send
  `categoryId`/`accountId`; if either does not match, return the field error without calling the
  API. Map a 422 `NotFound` violation to `{ field, message }` with `violationMessage` (T004) ·
  skill: frontend-developer
- [X] T035 [US3] In `fe/app/ui/settings/accounts-panel.tsx` and the phone accounts tab in
  `fe/app/ui/phone/phone-settings.tsx`, change the empty text to "You have no accounts yet. Add one
  to start recording spendings." (add it to `messages.ts`) · skill: frontend-developer

**Checkpoint**: manual entry never creates items, on desktop and phone, enforced by the server.

---

## Phase 6: User Story 4 — An unknown category or account in a CSV is flagged, not created (Priority: P1)

**Goal**: The import review flags unknown categories and accounts as invalid fields; adding never
creates either; the server refuses unknown names per row.

**Independent Test**: quickstart §3.6 and §4.5–4.6.

### Backend tests (write first, see them fail)

- [X] T036 [P] [US4] Update `…/test/controller/SpendingImportsIT.kt`:
  - 6 rows, row 2 `categoryName: "Groceriess"`, row 5 `accountName: "Visa Gold"` → `422` with
    exactly the violations `#/spendings/2/categoryName` (`NotFound`, "is not one of your
    categories") and `#/spendings/5/accountName` (`NotFound`, "is not one of your accounts");
    no spendings, categories, accounts or import receipt created;
  - names in mixed case and with spaces match existing items;
  - remove the "creates each missing name once" cases;
  - `SpendingImportPerformanceIT` (1,000 rows) still passes within its limit · skill:
    backend-developer

### Backend implementation

- [X] T037 [US4] Replace `NamedItemResolver.resolveNames` with a lookup that loads all of the
  caller's categories and accounts once (one query each), keys them with `key()`, and returns
  `NameMaps` plus violations for every row whose name is missing
  (`#/spendings/{i}/categoryName`, `field = "spendings[$i].categoryName"`, and the account
  equivalent). In `…/main/service/SpendingImportService.kt#create`, run it after the rule checks
  and the idempotency replay, before the receipt insert, and throw one `ValidationException` with
  all name violations. (Done this way so a retried import that already succeeded still replays;
  rule violations are reported first, as before.) Update the KDoc ("creating each missing … once" →
  "every name must exist") · skill: backend-developer
- [X] T038 [US4] Delete `…/main/repository/NamedItemInserts.kt` (no callers remain after T027 and
  T037) and its tests, if any. Update the name field `@Schema` descriptions in
  `…/main/dto/spendingimport/SpendingImportRequest.kt` and add the `unknownNames` 422 example to
  `…/main/controller/SpendingImportController.kt`, as in contracts/openapi-delta.yaml. Run
  `./gradlew check`, then **Regenerate types** · skill: backend-developer

### Frontend tests (write first, see them fail)

- [X] T039 [P] [US4] Update `fe/app/ui/import/review-stage.test.tsx` and
  `fe/app/ui/phone/phone-review.test.tsx`:
  - rows "groceries" / "Groceriess" / unknown account / empty category → first valid and shown as
    "Groceries"; the other three fields `aria-invalid` with "Not one of your categories", "Not one
    of your accounts" and the existing empty message; badge "3 rows need attention"; "Add
    spendings" disabled;
  - fixing "Groceriess" to "groceries" makes it valid; suggestions list only existing items;
  - the phone Fix row screen shows the same messages · skill: frontend-developer
- [X] T040 [P] [US4] Update `fe/app/lib/actions.test.ts` (import): a 422 with
  `#/spendings/2/categoryName` code `NotFound` gives a row error
  `{ row: 2, field: 'category', message: 'Not one of your categories' }`; the same for
  `accountName` · skill: frontend-developer
- [X] T041 [P] [US4] Update `fe/e2e/import-csv.spec.ts` and the phone import spec: remove the case
  that expects new categories/accounts to be created; add the four-row file from the spec's
  Independent Test and check highlights, the disabled button, fixing, adding, and unchanged
  Settings counts · skill: frontend-developer

### Frontend implementation

- [X] T042 [US4] Pass `{ categories, accounts }` into `draftProblems` in
  `fe/app/ui/import/review-stage.tsx` and the review/Fix row helpers in `fe/app/lib/phone.ts`
  (already typed by T005); make sure the review's category/account suggestions come from the
  existing items only · skill: frontend-developer
- [X] T043 [US4] In `fe/app/lib/actions.ts#rowErrorsOf`, use `violationMessage(violation)` so
  `NotFound` maps to the unknown-name messages; the review then highlights the fields as with any
  server row error (spec edge case "Name removed meanwhile") · skill: frontend-developer

**Checkpoint**: all four stories work; nothing outside Settings creates categories or accounts.

---

## Phase 7: Polish & Cross-Cutting Concerns

- [X] T044 [P] Update `fe/README.md` and `hl-budget-backend/README.md` where they describe
  creating categories or accounts by name, or the donut colours · skill: none
- [X] T045 [P] Mark the replaced requirements in earlier specs with a short note pointing to 007:
  `specs/003-spending-screens/spec.md` FR-035 and FR-072, `specs/004-csv-import/spec.md` FR-032,
  `specs/006-mobile-view/spec.md` FR-048 and FR-096 (one line each, e.g. "Changed by 007: names
  must exist") · skill: none
- [X] T046 Run the full gates: `./gradlew check` in the backend; `pnpm typecheck && pnpm lint &&
  pnpm test && pnpm e2e` in the frontend (desktop and phone projects, axe) · skill: none
- [X] T047 Backend release first (research R-014): open the PR, merge after the user agrees, let
  the release workflow open the hl-helm PR, merge it after the user agrees, then verify with
  `homelab:hl-deploy-status` (V5 applied, pod Ready) · skill: homelab:hl-deploy-status
- [X] T048 Frontend release: the same flow, then `homelab:hl-deploy-status` · skill:
  homelab:hl-deploy-status
- [X] T049 Live check on https://budget.valyalkin.com on a desktop and a phone (quickstart §5:
  §4.1–4.3), then update the README Features table in the spec repo with a small PR (row for
  007, "Implemented <date>") · skill: none

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (T001)** → everything.
- **Foundational (T002–T005)** → US3 and US4 frontend tasks. US1 and US2 do not need it.
- **US1**: backend T006–T015 → frontend T019 onward (needs regenerated types from T015);
  frontend tests T016–T018 can be written as soon as T001 is done.
- **US2**: needs T019–T021 (slices carry `color`); otherwise independent.
- **US3**: backend T026–T027 are independent of US1's backend; frontend needs Phase 2.
- **US4**: T037 shares `NamedItemResolver.kt` with T027, so do T027 first; T038 needs both;
  frontend needs Phase 2.
- **Polish**: after the stories you intend to ship.

### User Story Dependencies

- US1 (colours) — none beyond setup. MVP.
- US2 (label) — US1's colour wiring.
- US3 (manual entry) — Phase 2.
- US4 (import) — Phase 2; backend after T027.

### Within Each User Story

Tests first and failing → backend model/migration → services → API → regenerate types → frontend
data → components → e2e.

### Parallel Opportunities

- T002 and T003 together; T006–T009 together; T016–T018 together; T023 with T024; T026 with
  T028–T031; T036 with T039–T041; T044 with T045.
- A backend track (T006–T015, T026–T027, T036–T038) can run alongside a frontend track
  (T002–T005, T016–T018, T023–T025 once T019–T021 land, T028–T035, T039–T043).

---

## Parallel Example: User Story 1

```text
Task: "T006 CategoryPaletteTest in …/test/model/CategoryPaletteTest.kt"
Task: "T007 CategoryColorMigrationIT in …/test/repository/CategoryColorMigrationIT.kt"
Task: "T008 colour cases in DefaultCategoriesIT / CategoryControllerIT"
Task: "T009 colour in TotalsIT"
Task: "T016 by-category-card colour tests"
Task: "T017 phone topCategories / category-breakdown colour tests"
```

## Parallel Example: backend and frontend tracks

```text
Developer A (backend): T006–T009 → T010 → T011 → T012 → T013 → T014 → T015 → T026 → T027 → T036 → T037 → T038
Developer B (frontend): T002, T003 → T004 → T005 → T016–T018 → (wait for T015) T019 → T020, T021 → T022 → T023, T024 → T025 → T028–T035 → T039–T043
```

---

## Implementation Strategy

### MVP First (User Story 1 + 2)

1. Phase 1, then US1 backend and frontend.
2. **Stop and validate**: quickstart §3.1–3.3 and §4.1.
3. Add US2 (label) — together they fix the reported pie bug and can be released on their own
   (the backend change is additive).

### Incremental Delivery

1. US1 + US2 → chart fixed; release if wanted.
2. Phase 2 + US3 → manual entry only uses existing items (server enforced).
3. US4 → imports only use existing items.
4. Polish → docs, gates, releases (backend before frontend), live check, README.
