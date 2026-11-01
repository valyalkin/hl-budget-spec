---

description: "Task list for 008-account-breakdown"
---

# Tasks: Spending by account

**Input**: Design documents from `/specs/008-account-breakdown/`

**Prerequisites**: plan.md, spec.md, research.md, data-model.md, contracts/openapi-delta.yaml,
quickstart.md

**Tests**: REQUIRED. Constitution Principle IV: tests ship with the code, and every new
user-visible flow gets a Playwright spec. Within each phase, write the tests first and see them
fail before writing the code that satisfies them.

**Organization**: Tasks are grouped by user story (spec.md US1–US3). All backend work (account
colours and account totals) is foundational, because every story reads it. The frontend
breakdown components are generalised in the foundation too, so each story only wires up its own
screen.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Can run in parallel (different files, no dependency on an unfinished task).
- **[Story]**: The user story the task belongs to (US1 … US3).
- Every task ends with `· skill: <name>`. Load that skill before starting; its definition of done
  is part of the task's done. Where a skill and the constitution disagree, the constitution wins.

## Path Conventions

All paths are relative to the spec repo root `/Users/valyalkin/Documents/Projects/hl-budget-spec/`.

- `hl-budget-backend/`: git worktree of `valyalkin/hl-budget-backend`, on branch
  `008-account-breakdown` (T001).
  - Main code: `hl-budget-backend/src/main/kotlin/com/valyalkin/hlbudget/`, written `…/main/`.
  - Tests: `hl-budget-backend/src/test/kotlin/com/valyalkin/hlbudget/`, written `…/test/`.
  - Migrations: `hl-budget-backend/src/main/resources/db/migration/`, written `…/migration/`.
- `hl-budget-frontend/`: git worktree of `valyalkin/hl-budget-frontend`, on branch
  `008-account-breakdown` (T001). Written `fe/`.
- **Commits**: inside the service repos, Conventional Commits (`feat:`, `test:`, `refactor:`),
  after each task or logical group.
- **Merging**: merging to `main` in a service repo triggers a release. Do it only after the gate
  is green and the user agrees.

Shared rules that several tasks reference:

- **Regenerate types**: run `./gradlew check` in `hl-budget-backend/`; `OpenApiExportIT` rewrites
  `hl-budget-backend/contracts/api.yaml`. Then
  `cp hl-budget-backend/contracts/api.yaml hl-budget-frontend/contracts/api.yaml` and, in `fe/`,
  `pnpm generate:api`. Commit both frontend files.
- **Account colour order** (research R-002, R-003): the tie order of `CategoryPalette` — spare
  `#e11d48`, `#0891b2`, `#a855f7`, `#365314`, `#1e3a8a`, `#701a75`, `#ea580c`, then the 17
  default colours `#1e5b45`, `#c2410c`, `#1d4ed8`, `#a16207`, `#7e22ce`, `#0f766e`, `#be185d`,
  `#4d7c0f`, `#b91c1c`, `#0369a1`, `#57534e`, `#4338ca`, `#64748b`, `#16a34a`, `#c026d3`,
  `#7c2d12`, `#8a6d00`. A new account gets `CategoryPalette.next(counts)` where `counts` are the
  caller's **accounts** per colour (categories are not counted).
- **Colour column rule** (data-model.md): "`varchar(7)` NOT NULL, default `#8a908b`",
  "`CHECK color ~ '^#[0-9a-f]{6}$'`", "Set once on create, never updated (entity field
  `updatable = false`)".
- **Not yours = not found** (backend): every new query filters on the caller's `userId`.
- **By category must not change** (FR-001): same heading, same localStorage key
  `hlb.byCategoryView`, same test ids (`category-swatch`, `legend-swatch`), same texts. The
  existing By category unit tests and e2e specs must pass unchanged, except for import paths.
- **Client components**: no `useMemo`, `useCallback` or `React.memo` (the React Compiler is on).

---

## Phase 1: Setup

**Purpose**: Fresh feature branches with green gates.

- [X] T001 In both `hl-budget-backend/` and `hl-budget-frontend/`:
  1. `git fetch origin` and `git switch -c 008-account-breakdown origin/main` (both worktrees are
     on `007-fix-pie-csv-categories`, already merged; if there is uncommitted work, ask the user).
  2. Confirm the gates are green before any change: `./gradlew check` in the backend;
     `pnpm install && pnpm typecheck && pnpm lint && pnpm test` in the frontend.

  In the spec repo, create branch `008-account-breakdown` from `main` and ask the user before
  committing `specs/008-account-breakdown/` · skill: none

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: Accounts have colours, the server returns totals per account, the frontend has the
new types and data, and the breakdown components accept a `kind`. No visible change yet.

### Backend tests first (see them fail)

- [X] T002 [P] Create `…/test/repository/AccountColorMigrationIT.kt`, modelled on
  `CategoryColorMigrationIT.kt`: migrate a fresh database to V5 with Flyway `target("5")`, insert
  a user with 9 accounts (distinct `created_at`) and a second user with 2, migrate to latest, and
  assert: user 1's accounts in creation order have `#e11d48`, `#0891b2`, `#a855f7`, `#365314`,
  `#1e3a8a`, `#701a75`, `#ea580c`, `#1e5b45`, `#c2410c`; user 2's have `#e11d48`, `#0891b2`; an
  `UPDATE accounts SET color = 'red'` fails the check constraint · skill: backend-developer
- [X] T003 [P] Extend `…/test/service/AccountServiceTest.kt`: `create` with existing account
  colours `{#e11d48: 1}` gives the new account `#0891b2`; with none gives `#e11d48`; `rename`
  keeps the colour; the returned `Account` carries `color`. Update every `AccountEntity(...)`
  constructor call in `AccountServiceTest.kt`, `SpendingServiceTest.kt`,
  `SpendingImportServiceTest.kt` and `NamedItemResolverTest.kt` for the new `color` parameter ·
  skill: backend-developer
- [X] T004 [P] Extend `…/test/controller/AccountControllerIT.kt`: a new user creates "Amex",
  "Debit card", "Cash" → `color` `#e11d48`, `#0891b2`, `#a855f7`; `GET /api/v1/accounts` and `GET
  /api/v1/accounts/{id}` include the same `color`; renaming Amex keeps `#e11d48`; deleting unused
  "Cash" then creating "Savings" gives `#a855f7`; another user's first account is `#e11d48`
  regardless of the first user's (quickstart §3.1–3.2) · skill: backend-developer
- [X] T005 [P] Extend `…/test/controller/TotalsIT.kt` with account-totals cases (quickstart
  §3.3–3.6):
  - October spendings: Amex 2 × (50.00, 126.44), Debit card 1 × 2150.00, plus a September Amex
    spending → `GET /api/v1/account-totals?from=2026-10-01&to=2026-10-31` returns `items` [Debit
    card 2150.00 count 1, Amex 176.44 count 2] in that order, each with its account's `color`;
    `total` 2326.44 equals the category-totals total and the October monthly total;
    `spendingCount` 3;
  - an account with no spending in the range is absent; an empty range → `items: []`, total
    `0.00`;
  - equal totals are ordered by account name ignoring case;
  - `from` after `to` → 422 pointer `#/from`; 367 days → 422 pointer `#/to`; missing `from` →
    422; no token → 401;
  - a second user's spendings never appear.
  · skill: backend-developer
- [X] T006 [P] Extend `…/test/service/TotalsServiceTest.kt`: `byAccount` maps rows to
  `AccountTotal` with `Ref`, money in the app currency and `color`, sums `total` and
  `spendingCount`, and applies the same range checks as `byCategory` · skill: backend-developer
- [X] T007 [P] Extend `…/test/controller/TotalsPerformanceIT.kt`: with the existing 10,000
  spendings, `GET /api/v1/account-totals` for one month answers within the same limit as the
  category-totals check (SC-002) · skill: backend-developer

### Backend implementation

- [X] T008 Create `…/migration/V6__add_account_colors.sql` (research R-003): header comment like
  V5's; `ALTER TABLE accounts ADD COLUMN color varchar(7) NOT NULL DEFAULT '#8a908b';` then a
  `WITH tie_order (idx, color) AS (VALUES (0,'#e11d48'), … (23,'#8a6d00'))` in the shared order,
  `numbered AS (SELECT id, row_number() OVER (PARTITION BY user_id ORDER BY created_at, id) AS rn
  FROM accounts)`, `UPDATE accounts a SET color = t.color FROM numbered n JOIN tie_order t ON t.idx
  = (n.rn - 1) % 24 WHERE a.id = n.id;` and finally `ALTER TABLE accounts ADD CONSTRAINT
  ck_accounts_color CHECK (color ~ '^#[0-9a-f]{6}$');`. Keep the default (rolling deploys) ·
  skill: backend-developer
- [X] T009 Carry `color` through the account layers:
  - `…/main/repository/account/AccountEntity.kt`: `@Column(name = "color", nullable = false,
    updatable = false, length = 7) var color: String`, placed after `updatedAt`;
  - `…/main/repository/account/AccountRepository.kt`: `AccountRow` gains `color`; both counted
    queries select `a.color`; add `interface AccountColorCount { val color: String; val count:
    Long }` (or reuse `ColorCount` from the category repository) and `@Query("SELECT a.color AS
    color, count(a) AS count FROM AccountEntity a WHERE a.userId = :userId GROUP BY a.color") fun
    colorCounts(userId: UUID)`;
  - `…/main/repository/account/AccountMappings.kt`, `…/main/model/Account.kt` (+ `color: String`
    with KDoc "`#rrggbb`, assigned on create (feature 008)"), `…/main/dto/account/AccountMappings.kt`;
  - `…/main/dto/account/AccountResponse.kt`: `@field:Schema(description = "The account's colour,
    assigned when it is created and kept for its lifetime (including renames)", example =
    "#e11d48", pattern = "^#[0-9a-f]{6}$", accessMode = Schema.AccessMode.READ_ONLY) val color:
    String`.
  · skill: backend-developer
- [X] T010 In `…/main/service/AccountService.kt` `create`, set `color =
  CategoryPalette.next(accounts.colorCounts(userId).associate { it.color to it.count.toInt() })`
  with a comment pointing to research R-002, and return it in `Account`. Update the KDoc of
  `…/main/model/CategoryPalette.kt` to say accounts use the same palette and `next`, counted over
  the user's accounts only · skill: backend-developer
- [X] T011 Add account totals:
  - `…/main/repository/spending/TotalsQueries.kt`: `data class AccountTotalRow(accountId,
    accountName, accountColor, total, spendingCount)` and `fun byAccount(userId, from, to)` with
    the SQL from data-model.md, ordered `total DESC, lower(a.name), a.id`;
  - `…/main/model/Totals.kt`: `AccountTotal(account: Ref, total: Money, spendingCount: Long,
    color: String)` and `AccountTotals(from, to, total, spendingCount, items)`;
  - `…/main/service/TotalsService.kt`: `byAccount(identity, from, to)` mirroring `byCategory`
    (same `checkRange`); update the class KDoc to mention By account;
  - `…/main/dto/totals/TotalsResponses.kt`: `@Schema(name = "AccountTotal")
    AccountTotalResponse(account: RefDto, total, spendingCount, color)` and `@Schema(name =
    "AccountTotals") AccountTotalsResponse`, descriptions as in `contracts/openapi-delta.yaml`,
    plus `AccountTotals.toResponse()`;
  - `…/main/controller/TotalsController.kt`: `@GetMapping("/api/v1/account-totals")` operation
    `listAccountTotals`, summary, description, parameters, responses (200 with an `october`
    example including `color`, 401, 422, 500, 503) exactly as in the contract delta; class KDoc
    "by month, by category and by account". Also add the missing `color` to the existing
    `CATEGORY_EXAMPLE` items.
  · skill: backend-developer
- [X] T012 Run `./gradlew check` (T002–T007 now pass; Kover floors hold), then **Regenerate
  types** (see shared rules). Check the exported `contracts/api.yaml` contains `listAccountTotals`,
  `AccountTotals`, `AccountTotal` and `Account.color` · skill: backend-developer

### Frontend foundation tests first (see them fail)

- [X] T013 [P] Extend `fe/app/lib/data.test.ts`: `getAccountTotals({ year: 2026, month: 10 })`
  calls `/api/v1/account-totals?from=2026-10-01&to=2026-10-31`; `loadSpendingsPage` returns
  `accountTotals` and fetches it in the same parallel round as category totals · skill:
  frontend-developer
- [X] T014 [P] Rename `fe/app/lib/phone.test.ts` cases for `topCategories` to `topSlices`
  (behaviour unchanged: top 6 + "Everything else" in `#8a908b`) · skill: frontend-developer

### Frontend foundation implementation

- [X] T015 Add `AccountTotals` and `AccountTotal` aliases to `fe/app/lib/definitions.ts`; add
  `getAccountTotals(ym)` to `fe/app/lib/data.ts` (same shape as `getCategoryTotals`) and include
  it in `loadSpendingsPage`'s second `Promise.all` and in `SpendingsPageData` (research R-009) ·
  skill: frontend-developer
- [X] T016 Create `fe/app/ui/spendings/breakdown-kind.ts`: `export type BreakdownKind =
  'category' | 'account'`, `export type BreakdownItem = { id: string; name: string; cents: number;
  color: string }` and a `BREAKDOWN` record per kind with `heading` ("By category" / "By
  account"), `headingId` (`by-category` / `by-account`), `column` ("Category" / "Account"),
  `singular`/`plural` ("category"/"categories", "account"/"accounts"), `swatchTestId`
  (`category-swatch` / `account-swatch`), `legendTestId` (`legend-swatch` /
  `account-legend-swatch`) and `storageKey` (`hlb.byCategoryView` / `hlb.byAccountView`) · skill:
  frontend-developer
- [X] T017 Generalise the desktop components with `git mv` so history is kept (research R-006):
  - `by-category-card.tsx` → `fe/app/ui/spendings/breakdown-card.tsx` (`BreakdownCard` with a
    `kind` prop; heading, `aria-labelledby` id and storage key from `BREAKDOWN[kind]`; the
    module-level view store becomes one store per storage key, so the two cards never share a
    choice);
  - `category-table.tsx` → `breakdown-table.tsx` (`BreakdownTable`, column header and swatch test
    id from `kind`);
  - `category-pie.tsx` → `breakdown-pie.tsx` (`BreakdownPie`; summary "Spending by account: 3
    accounts, largest Amex at 38%" built from `singular`/`plural`; legend test id from `kind`);
  - `by-category-card.test.tsx` → `breakdown-card.test.tsx`, existing cases rendering
    `kind="category"`;
  - `fe/app/ui/spendings/month-section.tsx` renders `<BreakdownCard kind="category" …>` in place
    of `ByCategoryCard`.
  By category must look and behave exactly as before (shared rule) · skill: frontend-developer
- [X] T018 Generalise the phone breakdown (research R-007): `git mv
  fe/app/ui/phone/category-breakdown.tsx fe/app/ui/phone/breakdown.tsx` (and its test) as
  `Breakdown` with `kind`, `value` and `onChange` props for the Table / Pie choice instead of
  reading `ui.chart` itself; table `aria-label` "Spending by category" / "Spending by account";
  donut summary "Donut chart of [label] spending by account. Values are listed below."; swatch
  test id from `kind`. Rename `topCategories` → `topSlices` and `CategorySlice` → `Slice` in
  `fe/app/lib/phone.ts`. `fe/app/ui/phone/phone-app.tsx` renders `<Breakdown kind="category"
  value={ui.chart} onChange={ui.setChart} …>` · skill: frontend-developer
- [X] T019 Run `pnpm typecheck && pnpm lint && pnpm test` and `pnpm e2e --grep "by category"`:
  all By category specs pass unchanged · skill: frontend-developer

**Checkpoint**: the backend serves account colours and totals; the frontend loads them; nothing
new is on screen and By category is unchanged.

---

## Phase 3: User Story 1 — See the month by account on the desktop (Priority: P1) 🎯 MVP

**Goal**: A By account card below By category with its own Table / Pie toggle.

**Independent Test**: With October spendings on three of four accounts, the By account table
lists those three with amount, share and a total equal to the month total; Pie shows three
slices with a legend.

### Tests (write first, see them fail)

- [X] T020 [P] [US1] Add `kind="account"` cases to `fe/app/ui/spendings/breakdown-card.test.tsx`:
  heading "By account"; table column "Account"; rows largest first with whole-number shares and a
  total row; empty items → "Nothing spent in October 2026."; choosing Pie on the account card
  writes `hlb.byAccountView` and leaves `hlb.byCategoryView` untouched; two cards rendered
  together keep independent choices · skill: frontend-developer
- [X] T021 [P] [US1] Create `fe/e2e/by-account.spec.ts` (desktop project), seeding accounts and
  spendings the way `fe/e2e/by-category.spec.ts` does:
  - the By account card is directly below By category in the "Month summary" region;
  - Table lists only accounts with October spending, largest first, total row equals the month
    total card;
  - Pie shows one slice per account and a legend with name, amount and share; no "Everything
    else";
  - choose Pie on By account and Table on By category → after a month change and a reload both
    choices are kept;
  - edit a spending from Amex to Debit card → both totals change without a page reload (FR-008);
  - an empty month shows the empty message;
  - at a width where the summary cards drop below the table, By account is still below By
    category.
  · skill: frontend-developer

### Implementation

- [X] T022 [US1] In `fe/app/ui/spendings/month-section.tsx`, render `<BreakdownCard
  kind="account" label={label} currency={currency} totalCents={toCents(data.accountTotals.total.amount)}
  items={data.accountTotals.items.map((item) => ({ id: item.account.id, name: item.account.name,
  cents: toCents(item.total.amount), color: item.color }))} />` directly after the By category
  card inside the `aside` (FR-010) · skill: frontend-developer
- [X] T023 [US1] Run `pnpm test` and `pnpm e2e --grep "by account"` (desktop) until T020–T021
  pass · skill: frontend-developer

**Checkpoint**: US1 works on the desktop; can be released on its own.

---

## Phase 4: User Story 2 — Each account has its own colour and named slices (Priority: P1)

**Goal**: Account slices, legend and table swatches use the account's stored colour, the same
every month, and a slice names its account on hover, focus or tap.

**Independent Test**: Four accounts with spendings in two months → four different colours that
match legend and table swatches, the same in both months; hovering each slice shows "Name ·
amount · share".

The colours come from the backend (Phase 2) and the label from the shared `DonutChart` (feature
007), so this phase is tests that prove it for accounts, plus any fix they expose.

### Tests (write first)

- [X] T024 [P] [US2] In `fe/app/ui/spendings/breakdown-card.test.tsx`, for `kind="account"`:
  each `account-swatch` and `account-legend-swatch` has its item's `color` as background, and
  each slice path has it as `fill`; hovering a slice shows "Amex · $1,240.00 · 38%"; focusing a
  slice exposes the same text to assistive technology · skill: frontend-developer
- [X] T025 [P] [US2] Extend `fe/e2e/by-account.spec.ts`: four accounts with spendings in October
  and November → four different slice colours, each equal to its legend and table swatch, and
  each account's colour equal in both months; hover each slice → label with its name appears
  within 0.5 s (SC-006) and the slice is emphasised; Tab onto slices → same label; a newly created
  account "Savings" has a colour not used by the other four · skill: frontend-developer
- [X] T026 [P] [US2] Add the By account card, in Table and in Pie view, to the desktop scan in
  `fe/e2e/a11y.spec.ts` (no serious or critical issues) · skill: frontend-developer

### Implementation

- [X] T027 [US2] Run T024–T026; fix anything they expose in `fe/app/ui/spendings/breakdown-pie.tsx`
  or `breakdown-table.tsx` without changing By category · skill: frontend-developer

**Checkpoint**: US1 + US2 complete the desktop feature.

---

## Phase 5: User Story 3 — See the month by account on the phone (Priority: P2)

**Goal**: The phone switch reads "Spendings | By category | By account"; By account has its own
Table / Pie choice and a top-6 donut.

**Independent Test**: At 390 px wide with October spendings on 8 accounts, By account's table
lists all 8 with a total equal to the month total; Pie shows 6 slices plus "Everything else"
equal to the other two.

### Tests (write first, see them fail)

- [X] T028 [P] [US3] Extend `fe/app/ui/phone/phone-ui-provider.test.tsx`: `view` accepts
  `'by-account'`; `accountChart` defaults to `'table'`; `setAccountChart('pie')` leaves `chart`
  unchanged and vice versa; both survive the provider's month-change path · skill:
  frontend-developer
- [X] T029 [P] [US3] Extend `fe/app/ui/phone/month-year-picker.test.tsx`: the "Show" switch has
  three pressed-state buttons "Spendings", "By category", "By account"; pressing "By account"
  calls `setView('by-account')` · skill: frontend-developer
- [X] T030 [P] [US3] Extend `fe/app/ui/phone/breakdown.test.tsx` for `kind="account"`: table
  `aria-label` "Spending by account", highest first, "Total" row with 100%; Pie with 8 accounts →
  6 slices plus "Everything else" equal to the other two; with 6 or fewer → no "Everything else";
  empty → "Nothing to break down in October 2026."; tapping a slice shows its label · skill:
  frontend-developer
- [X] T031 [P] [US3] Create `fe/e2e/phone/by-account.spec.ts` (phone project), modelled on
  `fe/e2e/phone/by-category.spec.ts`: switch to By account; table and top-6 donut as in T030;
  choose Pie for By account and Table for By category → each kept, and both kept after a month
  change; empty month message; at 360 px wide the three switch options sit on one row, each at
  least 44 px tall, with no horizontal page scroll (FR-015). Add the By account view (Table and
  Pie) to `fe/e2e/phone/a11y.spec.ts` · skill: frontend-developer

### Implementation

- [X] T032 [US3] In `fe/app/ui/phone/phone-ui-provider.tsx`: `view: 'spendings' | 'by-category'
  | 'by-account'`; add `accountChart: 'table' | 'pie'` (default `'table'`) and `setAccountChart`
  next to `chart` · skill: frontend-developer
- [X] T033 [US3] In `fe/app/ui/phone/month-year-picker.tsx`, add `{ value: 'by-account', label:
  'By account' }` as the third switch option; update the component KDoc. In
  `fe/app/ui/phone/segmented.tsx`, use `px-2` instead of `px-3` when there are three or more
  options and allow the label to wrap (`leading-tight`), keeping `min-h-11` (research R-008) ·
  skill: frontend-developer
- [X] T034 [US3] Pass account totals to the phone: `fe/app/ui/phone/phone-data.ts` gains
  `accountTotals: { totalCents: number; items: BreakdownItem[] }`;
  `fe/app/ui/phone/phone-section.tsx` maps `data.accountTotals` like `categoryTotals`;
  `fe/app/ui/phone/test-support.tsx` fixtures gain `accountTotals`. In
  `fe/app/ui/phone/phone-app.tsx`, render `<Breakdown kind="account" value={ui.accountChart}
  onChange={ui.setAccountChart} …>` when `ui.view === 'by-account'` · skill: frontend-developer
- [X] T035 [US3] Run `pnpm test` and `pnpm e2e --project phone` until T028–T031 pass · skill:
  frontend-developer

**Checkpoint**: all three stories work on desktop and phone.

---

## Phase 6: Polish & Cross-Cutting Concerns

- [X] T036 [P] Update `fe/README.md` and `hl-budget-backend/README.md` where they list the
  Spendings screen's cards, the phone switch or the totals endpoints · skill: none
- [X] T037 [P] Add a one-line note to `specs/003-spending-screens/spec.md` (US5 / FR-070) and
  `specs/006-mobile-view/spec.md` (FR-017) pointing to 008 ("Extended by 008: a By account
  breakdown sits alongside") · skill: none
- [X] T038 Run the full gates: `./gradlew check` in the backend; `pnpm typecheck && pnpm lint &&
  pnpm test && pnpm e2e` in the frontend (desktop and phone projects, axe) · skill: none
- [X] T039 Backend release first (research R-010): open the PR, merge after the user agrees, let
  the release workflow open the hl-helm PR, merge it after the user agrees, then verify with
  `homelab:hl-deploy-status` (V6 applied, pod Ready) · skill: homelab:hl-deploy-status
- [X] T040 Frontend release: the same flow, then `homelab:hl-deploy-status` · skill:
  homelab:hl-deploy-status
- [X] T041 Live check on https://budget.valyalkin.com on a desktop and a phone (quickstart §5):
  existing accounts each have a colour (none grey `#8a908b`), By account totals equal the month
  total. Then update the README Features table in the spec repo with a small PR (row for 008,
  "Implemented <date>") · skill: none

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (T001)** → everything.
- **Foundational**: backend T002–T007 (tests) → T008 → T009 → T010 → T011 → T012. Frontend
  T013–T014 can be written right after T001; T015 needs T012's regenerated types; T016 → T017,
  T018 → T019.
- **US1** (T020–T023): needs Phase 2.
- **US2** (T024–T027): needs T022 (the account card on screen).
- **US3** (T028–T035): needs Phase 2 only; independent of US1 and US2.
- **Polish**: after the stories you intend to ship.

### User Story Dependencies

- US1 (desktop card) — Phase 2. MVP.
- US2 (colours and labels) — US1's card.
- US3 (phone) — Phase 2; can run in parallel with US1/US2.

### Within Each User Story

Tests first and failing → wiring → run the story's tests → checkpoint.

### Parallel Opportunities

- T002–T007 together (different test files).
- T013 and T014 together, alongside the backend implementation.
- T017 and T018 together after T016.
- T020 with T021; T024–T026 together; T028–T031 together.
- US3 (T028–T035) alongside US1 + US2 (T020–T027).
- T036 with T037.

---

## Parallel Example: Foundational backend tests

```text
Task: "T002 AccountColorMigrationIT in …/test/repository/AccountColorMigrationIT.kt"
Task: "T003 colour on create in …/test/service/AccountServiceTest.kt"
Task: "T004 colours in …/test/controller/AccountControllerIT.kt"
Task: "T005 account totals in …/test/controller/TotalsIT.kt"
Task: "T006 byAccount in …/test/service/TotalsServiceTest.kt"
Task: "T007 account totals in …/test/controller/TotalsPerformanceIT.kt"
```

## Parallel Example: desktop and phone tracks after Phase 2

```text
Developer A (desktop): T020, T021 → T022 → T023 → T024–T026 → T027
Developer B (phone):   T028–T031 → T032 → T033 → T034 → T035
```

---

## Implementation Strategy

### MVP First (User Story 1 + 2)

1. Phase 1, then Phase 2 (backend can be released on its own: the changes are additive).
2. US1: the desktop By account card.
3. **Stop and validate**: quickstart §3 and §4.1–4.5.
4. US2: prove colours and labels for accounts. Release the desktop feature if wanted.

### Incremental Delivery

1. Phase 2 → backend release (nothing visible changes).
2. US1 + US2 → By account on the desktop.
3. US3 → By account on the phone.
4. Polish → docs, gates, frontend release, live check, README.
