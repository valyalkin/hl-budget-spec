---

description: "Task list for 003-spending-screens"
---

# Tasks: Spending Screens (Web App)

**Input**: Design documents from `/specs/003-spending-screens/`

**Prerequisites**: plan.md, spec.md, research.md, data-model.md, contracts/openapi.yaml,
quickstart.md

**Tests**: REQUIRED. Constitution Principle IV says tests ship with the code and every new
user-visible flow gets a Playwright spec. Research R-013 maps each success criterion to a test.
Within each story, write the tests first and see them fail before writing the code that
satisfies them.

**Organization**: Tasks are grouped by user story (spec.md US1–US7), so each story can be built
and checked on its own. Each story phase does its backend part first, then regenerates the
frontend's API types, then builds the screens.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Can run in parallel (different files, no dependency on an unfinished task).
- **[Story]**: The user story the task belongs to (US1 … US7).
- Every task ends with `· skill: <name>`. Load that skill before starting the task; its
  definition of done is part of the task's done (plan.md "Skills that drive the
  implementation"). Where a skill and the constitution disagree, the constitution wins
  (plan.md Complexity Tracking).

## Path Conventions

All paths are relative to the spec repo root `/Users/valyalkin/Documents/Projects/hl-budget-spec/`.

- `hl-budget-backend/`: the git worktree of `valyalkin/hl-budget-backend`, on branch
  `003-spending-screens` (T001).
  - Main code: `hl-budget-backend/src/main/kotlin/com/valyalkin/hlbudget/`, written below as
    `…/main/`.
  - Tests: `hl-budget-backend/src/test/kotlin/com/valyalkin/hlbudget/`, written below as
    `…/test/`.
- `hl-budget-frontend/`: the git worktree of `valyalkin/hl-budget-frontend`, on branch
  `003-spending-screens` (T001). Written below as `fe/`.
- **Commits**: commit inside the service repos (this spec repo ignores them) using
  Conventional Commits, after each task or logical group.
- **Merging**: merging to `main` in a service repo triggers a release. Do it only after the gate
  is green and the user agrees.

Shared rules that several tasks reference:

- **Regenerate types** (R-014): run `./gradlew check` in `hl-budget-backend/`, which makes
  `OpenApiExportIT` rewrite `hl-budget-backend/contracts/api.yaml`. Then
  `cp hl-budget-backend/contracts/api.yaml hl-budget-frontend/contracts/api.yaml`, and in `fe/`
  run `pnpm generate:api`. Commit both frontend files.
- **Money in the UI** (R-008): amounts are decimal strings. Parse them only with `toCents` from
  `fe/app/lib/utils.ts` into integer cents. Never use `parseFloat` or `Number(amount)` on money.
  Format only with `formatMoney`.
- **Messages** (R-007): every user-facing error string comes from `fe/app/lib/messages.ts`,
  worded exactly as in spec.md.
- **Server Actions** (R-009): each action lives in `fe/app/lib/actions.ts` (`'use server'`).
  1. It re-checks the session; with no session it returns `{ status: 'signed-out' }`.
  2. It validates input with Zod.
  3. It makes **exactly one** API call through `authedFetch`.
  4. On success it calls `refresh()` from `next/cache`.
  5. It returns `{ status: 'ok', … }` or `{ status: 'error', message, field? }`. It never throws
     to the client.
  6. Every client caller treats `{ status: 'signed-out' }` the same way: it calls
     `window.location.assign('/')`, so the unsaved row is dropped and the signed-out screen
     shows (spec Edge Cases "Session expiry").
- **Client components**: `'use client'` only on the files the plan marks. No `useMemo`,
  `useCallback` or `React.memo` (the React Compiler is on). The session and token never cross
  into props.
- **Not yours = not found** (backend, as in 002): every new query filters on the caller's
  `userId`.

---

## Phase 1: Setup

**Purpose**: Fresh feature branches with green gates, and the one new dev dependency.

- [X] T001 In both `hl-budget-backend/` and `hl-budget-frontend/`:
  1. Run `git fetch origin` and `git switch -c 003-spending-screens origin/main`.
  2. Confirm `./gradlew check` (backend) and `pnpm install && pnpm typecheck && pnpm lint && pnpm test` (frontend) are green before any change.

  Operator note for the user: the spec repo itself is on `docs/readme-002`; ask whether to commit `specs/003-spending-screens/` there or on a new `003-spending-screens` branch · skill: none
- [X] T002 [P] In `fe/`, add the dev dependency: `pnpm add -D @axe-core/playwright` (exact version pinned by the lockfile). Commit `fe/package.json` and `fe/pnpm-lock.yaml` · skill: frontend-developer

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: The frontend's shell, routes, error plumbing, money helpers and design tokens, and
the shared backend pieces. Every story needs them.

**⚠️ CRITICAL**: No user-story work starts until this phase is complete.

### Backend foundation

- [X] T003 [P] Create `…/main/dto/MoneyTotalDto.kt`: `data class MoneyTotalDto(val amount: String, val currency: String)` with `@Schema(name = "MoneyTotal", description = "A sum of amounts. Unlike Money it may be zero and has no 8-digit cap. Always 2 decimals.")`, `amount` with `pattern = "^\\d+\\.\\d{2}$"`, example `"2748.42"`, and `currency` with `pattern = "^[A-Z]{3}$"`. Add `fun BigDecimal.toMoneyTotal(currency: Currency) = MoneyTotalDto(setScale(2).toPlainString(), currency.currencyCode)` · skill: backend-developer
- [X] T004 [P] In `…/main/config/OpenApiConfig.kt`, add the tags `Totals` ("Sums of the caller's spendings by month and by category") and `Merchants` ("Merchants derived from the caller's spendings, for autofill") · skill: backend-developer

### Frontend foundation: tests first (see them fail)

- [X] T005 [P] Extend `fe/app/lib/api.test.ts` (MSW). For a problem body `{ type, traceId, details: { reason: "in-use", spendingCount: 3 }, errors: [{ field: "/date", message: "…" }] }`, `ApiError` exposes `status`, `type`, `traceId`, `reason === "in-use"`, `details.spendingCount === 3` and `errors[0].field === "/date"`. A body without `details` or `errors` leaves them `undefined`. The existing 401 and timeout behaviour is unchanged · skill: frontend-developer
- [X] T006 [P] Create `fe/app/lib/utils.test.ts` cases (extend the existing file):
  - **`toCents`**: `"84.37"` → `8437`, `"0.00"` → `0`, `"2150"` → `215000`, `"6.5"` → `650`. It throws on `"abc"`, `"1.234"` and `"-1"`.
  - **`centsToAmount`**: `8437` → `"84.37"`.
  - **`formatMoney`** (`{amount, currency}`, `opts?: { wholeUnits?: boolean }`): `"84.37"/CAD` → `"$84.37"` in `en-CA`; `wholeUnits` gives `"$2,748"`.
  - **`sharePercent`**: `(215000, 274842)` → `78`, and `0` when the total is 0.
  - **`parseMonthParam`** (`(value, today)`): `"2026-10"` → `{ year: 2026, month: 10 }`. A missing value, `"2026-13"`, `"abc"`, and a month after `today`'s all give `today`'s month.
  - **`monthBounds`**: `{2026, 2}` → `{ from: "2026-02-01", to: "2026-02-28" }`; 2028 gives `"2028-02-29"`.
  - **`monthLabel`**: `{2026, 10}` → `"October 2026"` · skill: frontend-developer
- [X] T007 [P] Create `fe/app/lib/messages.test.ts`. `messageFor(error, context)` maps:
  - 409 `duplicate-name` with `name: "Coffee"` → `“Coffee” already exists.`
  - 409 `in-use` with `spendingCount: 3` for a category named `Coffee` → `Coffee is used by 3 spendings, so it can’t be removed.`; with a count of 1, `1 spending`
  - the same for an account named `Visa` → `Visa has 3 spendings, so it can’t be removed.`
  - 422 `last-account` → `You need at least one account.`
  - 422 with an `errors` entry on `/date` → `Pick a date that is today or earlier.`
  - 404 → `This item no longer exists.`
  - `ApiUnavailableError` or any 5xx → `Couldn’t reach the server. Your changes weren’t saved.`

  Also cover the local validation strings, exported as constants: `Pick a date.`, `Add a merchant name.`, `Choose or type a category.`, `Choose or type an account.`, `Enter an amount greater than zero.`, `Type a name for the category.`, `Type a name for the account.` · skill: frontend-developer
- [X] T008 [P] Create `fe/app/ui/app-header.test.tsx` (RTL). Signed out, it renders the app name "HL Budget" and a **plain `<a href="/auth/login">`** "Sign in" (never a prefetching `Link`). Signed in, it renders "Sign out" (`<a href="/auth/logout">`) and a user-icon **button** with `aria-label="Settings"` that calls the `onOpenSettings` slot (rendered by a client child). There are no other links · skill: frontend-developer
- [X] T009 [P] Extend `fe/proxy.test.ts`:
  - `/spendings` and `/spendings?month=2026-10` without a session → 302 to `/auth/login?returnTo=%2Fspendings%3Fmonth%3D2026-10`.
  - With a session, `Cache-Control: private, no-cache, no-store`.
  - `/` is not gated.
  - `/home` is no longer in `PROTECTED_PREFIXES`.

  Extend `fe/app/seo.test.ts`: `/` and `/spendings` export `robots: { index: false, follow: false }`, and the sitemap lists neither · skill: frontend-developer
- [X] T010 [P] Replace `fe/app/ui/landing/*.test.tsx` with `fe/app/signed-out.test.tsx`. The signed-out screen shows the header, the heading "You're signed out", the text "Sign in to see and record your spendings." and a "Sign in" `<a href="/auth/login?returnTo=/spendings">`. With a session, the page calls `redirect('/spendings')` (mock `next/navigation`) · skill: frontend-developer

### Frontend foundation: implementation

- [X] T011 In `fe/app/lib/definitions.ts`, extend `ApiError` with `readonly reason?: string`, `readonly details?: Record<string, unknown>` and `readonly errors?: ReadonlyArray<{ field: string; message: string }>` (constructor takes an options object; keep `status, type, traceId` positional for existing callers). In `fe/app/lib/api.ts`, read `details` and `errors` from the problem body, set `reason` from `details.reason` when it is a string, and type-guard every value (`unknown` in, never `any`). Make T005 green · skill: frontend-developer
- [X] T012 [P] Implement the T006 helpers in `fe/app/lib/utils.ts`:
  - `toCents(amount: string): number` uses the regex `^\d+(\.\d{1,2})?$` and integer maths only.
  - `formatMoney` uses `Intl.NumberFormat('en-CA', { style: 'currency', currency })`, formatting from cents divided by 100 only at the formatting call.
  - `sharePercent(partCents, totalCents)` is `Math.round(part * 100 / total)`.
  - Also implement `parseMonthParam`, `monthBounds`, `monthLabel` and `isFutureMonth(ym, today)`.

  Make T006 green · skill: frontend-developer
- [X] T013 [P] Create `fe/app/lib/messages.ts` with `messageFor(error: unknown, context: { resource?: 'category' | 'account'; name?: string })` and the exported constants from T007. Make T007 green · skill: frontend-developer
- [X] T014 [P] Apply the design tokens (R-012):
  - **`fe/app/globals.css`**: a Tailwind 4 `@theme` block.
    - Colours: `--color-ground: #F3F4EF`, `--color-surface: #FFFFFF`, `--color-ink: #17191A`, `--color-ink-2: #3F4541`, `--color-muted: #5C625E`, `--color-line: #DDE0D8`, `--color-line-soft: #ECEEE8`, `--color-brand: #1E5B45`, `--color-brand-strong: #123B2C`, `--color-brand-tint: #E4EEE8`, `--color-error: #A33A16`, `--color-disabled: #8A908B`.
    - Fonts: `--font-display`, `--font-sans`, `--font-mono`.
  - **`fe/app/ui/fonts.ts`**: replace Inter with `next/font/google` `Bricolage_Grotesque` (weights 500 and 700, `variable: '--font-display'`), `IBM_Plex_Sans` (400, 500, 600; `--font-sans`) and `IBM_Plex_Mono` (400, 500; `--font-mono`), all `display: 'swap'`.
  - **`fe/app/layout.tsx`**: put the three font variables on `<html>`, and `bg-ground text-ink font-sans` on `<body>`.

  The CSP in `fe/next.config.ts` stays unchanged (`font-src 'self'`) · skill: frontend-developer
- [X] T015 Create `fe/app/ui/app-header.tsx`: a server component taking `signedIn: boolean` and an optional `settingsSlot: React.ReactNode`. It renders `<header>` with "HL Budget" in `font-display`, then either the Sign in link or Sign out + `settingsSlot`, styled per the design (brand button for Sign in, ghost link for Sign out). Delete `fe/app/ui/home/` and `fe/app/ui/landing/` with their tests once nothing imports them. Make T008 green · skill: frontend-developer
- [X] T016 Routes:
  - **`fe/proxy.ts`**: `PROTECTED_PREFIXES = ['/spendings']`.
  - **`fe/next.config.ts`**: add `async redirects()` with `{ source: '/home', destination: '/spendings', permanent: true }` and the same for `/home/:path*`.
  - **`fe/app/home/`**: delete it (page, loading, error and their tests).
  - **`fe/app/page.tsx`**: the signed-out screen. Keep the page synchronous. A `<Suspense>` child reads `getSession()`: with a session it calls `redirect('/spendings')`, otherwise it renders `<AppHeader signedIn={false} />` and the T010 content. `metadata`: `title: { absolute: 'HL Budget' }` and `robots: { index: false, follow: false }`.
  - **`fe/app/sitemap.ts`**: make sure `/` is not listed.
  - **`fe/app/robots.ts`**: keep only what the T009 SEO test expects.

  Make T009 and T010 green · skill: frontend-developer
- [X] T017 [P] Add `SpendingsSkeleton` to `fe/app/ui/skeletons.tsx`, mirroring the month line (12 cells), two cards and a table of 6 rows, with `aria-busy="true"` and the label "Loading spendings". Remove `ProfileCardSkeleton` and `LandingActionsSkeleton` if they are now unused · skill: frontend-developer
- [X] T018 [P] Extend `fe/mocks/handlers.ts` with default MSW handlers that later tasks override per test: `GET /api/v1/monthly-totals`, `/category-totals`, `/merchants`, `/spendings`, `/categories` and `/accounts`, using the contract examples in `specs/003-spending-screens/contracts/openapi.yaml` and `specs/002-spending-tracking/contracts/openapi.yaml` · skill: frontend-developer
- [X] T019 [P] Extend `fe/e2e/support.ts` with:
  - `signIn(page)`, the existing flow now landing on `/spendings`;
  - `addSpendingViaUi(page, { date, merchant, category, account, amount })`;
  - `removeSpendingsCreatedBy(page, marker)`, which deletes every row whose merchant contains a unique per-run marker (for example `e2e-<timestamp>`), so specs clean up their own data · skill: frontend-developer

**Checkpoint**: `pnpm typecheck && pnpm lint && pnpm test` is green. Signed out, `/` shows the
new header and the signed-out message. `/home` redirects to `/spendings` (404 for now).

---

## Phase 3: User Story 1 — See my spendings for a month (Priority: P1) 🎯 MVP

**Goal**: A signed-in user opens `/spendings` and sees the year and month line with totals, the
month's spendings in a table with a count and total footer, and the month total card, or the
empty message.

**Independent Test**: With spendings recorded through the API in September and October of the
current year, open `/spendings` and check that October is selected, and that its table, footer
and card match the data. Click September and check the same. Months after today's month are
disabled.

### Tests for User Story 1 ⚠️ (write first, see them fail)

- [X] T020 [P] [US1] Write `…/test/service/TotalsServiceTest.kt` (MockK) for `monthly(identity, year?)`:
  - It returns exactly 12 `MonthTotal`s for January–December, filling months missing from the query with `0.00` and count 0.
  - `year` null means the current year in `HlBudgetProperties.zoneId` from the injected `Clock`.
  - `today` is `LocalDate.now(clock.withZone(zoneId))`. At a fixed `Clock` of `2026-11-01T03:00:00Z` in `America/Toronto`, `today` is `2026-10-31` · skill: backend-developer
- [X] T021 [P] [US1] Write `…/test/controller/MonthlyTotalsIT.kt` (Testcontainers):
  - **Shape and sums**: `GET /api/v1/monthly-totals?year=2026` gives 200 with `year`, `today` and 12 `months` in order; amounts are exact decimal strings (for example `"84.37" + "16.99"` → `"101.36"`), `currency: "CAD"`, and correct counts.
  - **Defaults and errors**: no `year` gives the current year. `year=1999` or `year=2101` gives 422 with pointer `year`.
  - **Ownership**: another user's spendings never count.
  - **Auth**: 401 without a token · skill: backend-developer
- [X] T022 [P] [US1] Write `fe/app/lib/data.test.ts` cases (MSW):
  - `getMonthlyTotals(year?)` calls `/api/v1/monthly-totals` with `year` only when given, and returns the typed body.
  - `getMonthSpendings({ year, month })` calls `/api/v1/spendings?from=<first>&to=<last>&size=100&sort=date,desc&page=0`. When `totalPages` is 3, it fetches pages 1 and 2 and returns all items in order.
  - A 401 redirects to `/auth/login?returnTo=/spendings` (as `ensureMyProfile` does).
  - Other errors propagate · skill: frontend-developer
- [X] T023 [P] [US1] Write `fe/app/ui/spendings/month-line.test.tsx`:
  - It shows the year and 12 months, each with `formatMoney(total, { wholeUnits: true })`, or `—` when the count is 0.
  - The selected month has `aria-current="date"`.
  - Past months are links to `?month=YYYY-MM`.
  - Future months (after `today`) are `<button disabled>` with `aria-disabled="true"`.
  - The previous-year arrow links to `?month=<year-1>-12`.
  - The next-year arrow is disabled when the shown year is `today`'s year; otherwise it links to the next year's December, or `today`'s month when that year is the current one.
  - Both arrows have `aria-label`s ("Previous year" / "Next year") · skill: frontend-developer
- [X] T024 [P] [US1] Write `fe/app/ui/spendings/spendings-table.test.tsx` (read-only part):
  - Columns Date, Merchant, Category, Account, Amount in that order.
  - Rows in the given order; the date is shown as `Oct 27`; amounts are right-aligned in `font-mono` with tabular numbers.
  - Long text is truncated with the full value in `title`.
  - The footer shows `14 spendings` (`1 spending` for one) and the month total.
  - An empty month shows exactly `No spendings recorded in October 2026.` and no footer row.

  Also write `fe/app/ui/spendings/month-total-card.test.tsx`: the total and `N spendings` · skill: frontend-developer
- [X] T025 [P] [US1] Write `fe/e2e/month-view.spec.ts`:
  - Signed in, `/spendings` selects today's month, and future months are disabled.
  - Seed two spendings via the UI helpers, or the API helper if T019 provides one. Check the table rows, footer count and total, and the card.
  - Go to an empty past month and check the empty message.
  - Reloading keeps `?month=`.
  - Clean up afterwards · skill: frontend-developer

### Implementation for User Story 1

- [X] T026 [P] [US1] Create `…/main/model/MonthTotal.kt` (`month: YearMonth`, `total: BigDecimal`, `spendingCount: Long`), plus `…/main/dto/totals/MonthlyTotalsResponse.kt` (`@Schema(name = "MonthlyTotals")`: `year`, `today`, `months: List<MonthTotalResponse>` with `@Schema(name = "MonthTotal")`, `month` formatted `yyyy-MM`) and `…/main/dto/totals/TotalsMappings.kt` · skill: backend-developer
- [X] T027 [US1] Create `…/main/repository/spending/TotalsQueries.kt` (`JdbcClient`, `@Repository`). `monthly(userId, from, to)` runs `SELECT date_trunc('month', spending_date)::date AS m, sum(amount) AS total, count(*) AS n FROM spendings WHERE user_id = :userId AND spending_date BETWEEN :from AND :to GROUP BY 1`, mapping `total` as `BigDecimal` (never `Double`) · skill: backend-developer
- [X] T028 [US1] Create `…/main/service/TotalsService.kt` (`@Transactional(readOnly = true)`, resolving the caller with `CurrentUserService.resolve`). `monthly(identity, year: Int?)` fills empty months and computes `today` from `Clock` and `zoneId`. Make T020 green · skill: backend-developer
- [X] T029 [US1] Create `…/main/controller/TotalsController.kt`: `GET /api/v1/monthly-totals`, operationId `listMonthlyTotals`, tag `Totals`, `@RequestParam(required = false) @Min(2000) @Max(2100) year: Int?`. Document responses 200 (with the contract example), 401, 422, 500 and 503 using the existing shared refs. Make T021 green, then **regenerate types** · skill: backend-developer
- [X] T030 [US1] In `fe/app/lib/definitions.ts`, add the aliases `MonthlyTotals`, `MonthTotal`, `Spending`, `SpendingPage` and `MoneyTotal`, and the UI type `YearMonth = { year: number; month: number }`. In `fe/app/lib/data.ts`, add `getMonthlyTotals` and `getMonthSpendings`. Move the `ensureMyProfile()` call into a new `loadSpendingsPage(monthParam: string | undefined)`:
  1. In parallel: `ensureMyProfile()` and `getMonthlyTotals(yearOf(monthParam))`, where `yearOf` returns the param's year only if it is a valid `YYYY-MM`, otherwise `undefined` (current year).
  2. Resolve `ym = parseMonthParam(monthParam, totals.today)`. If `ym.year` differs from `totals.year` (a future or invalid param was clamped), fetch `getMonthlyTotals(ym.year)` once more.
  3. In parallel: `getMonthSpendings(ym)` plus the reads later stories add.

  It returns `{ ym, today, totals, spendings, … }`. Test in `fe/app/lib/data.test.ts`: a normal `?month=2026-09` makes exactly one `/monthly-totals` call; `?month=2031-01` makes two and returns today's month. Make T022 green · skill: frontend-developer
- [X] T031 [P] [US1] Create `fe/app/ui/spendings/month-line.tsx` (server component, `next/link` links, styled per the design with the selected month on `brand-tint`). Make T023 green · skill: frontend-developer
- [X] T032 [P] [US1] Create `fe/app/ui/spendings/month-total-card.tsx`. Create `fe/app/ui/spendings/spending-row.tsx` (a read-only `<tr>`) and `fe/app/ui/spendings/spendings-table.tsx` (`'use client'`, for now rendering rows, footer and empty state from props: `spendings`, `monthTotal`, `ym`, `today`). Make T024 green · skill: frontend-developer
- [X] T033 [US1] Create the route:
  - `fe/app/spendings/page.tsx`: synchronous; `metadata = { title: 'Spendings', robots: { index: false, follow: false } }`; renders `<AppHeader signedIn settingsSlot={…} />` (settings slot filled in US6) and `<Suspense fallback={<SpendingsSkeleton />}><MonthSection searchParams={props.searchParams} /></Suspense>`.
  - `fe/app/ui/spendings/month-section.tsx` (async): awaits `searchParams`, calls `loadSpendingsPage(searchParams.month)` once, and renders from its result. Lays out the month line, table and card per the design (two columns at ≥ 1024 px, stacked below).
  - `fe/app/spendings/loading.tsx` (the skeleton) and `fe/app/spendings/error.tsx` (`'use client'`, `reportClientError`, the text "We couldn't load your spendings." and a "Try again" button calling `reset`, per FR-092).
  - Make T025 green · skill: frontend-developer

**Checkpoint**: US1 is fully functional and testable on its own (quickstart §4 rows 1–3).

---

## Phase 4: User Story 2 — Record a spending in the table (Priority: P1)

**Goal**: The + button opens an editable row at the top. The user fills in date (calendar),
merchant, category and account (pick or type a new name) and amount, then saves with the check
mark or cancels with X. The view switches to the spending's month.

**Independent Test**: With one account, click +, fill every field, save, and check the spending
appears in the table and in the API with the exact values. Typing a new category name creates
it. Invalid input shows the spec's messages.

### Tests for User Story 2 ⚠️ (write first, see them fail)

- [X] T034 [P] [US2] Write `…/test/service/NamedItemResolverTest.kt` (MockK) for `resolveCategory(userId, ref)` and `resolveAccount(userId, ref)`:
  - **By id**: a ref given by id returns it when it is owned; otherwise it raises the 002 reference error (422 at `/categoryId`).
  - **By name, matching**: a name is trimmed; it must be non-empty and **at most 50 characters**, else a 422 `FieldViolation` at `/categoryName` or `/accountName`. A case-insensitive match is reused (`" coffee "` → the existing `Coffee`).
  - **By name, new**: with no match it is inserted, as `is_default = false` for categories.
  - **Race**: a `DataIntegrityViolationException` on insert (the race) is followed by a re-read that returns the winner · skill: backend-developer
- [X] T035 [P] [US2] Write `…/test/controller/SpendingByNameIT.kt`:
  - **Create and reuse**: `POST /api/v1/spendings` with `categoryName: "coffee"`, `accountName: "Visa"` when neither exists gives 201; `GET /categories` and `GET /accounts` now contain them. The same with `"COFFEE"` reuses the item, with no second category.
  - **One of each pair**: both `categoryId` and `categoryName` → 422 with pointer `/categoryId` and message "give exactly one of categoryId and categoryName". Neither → the same. The same pair of rules applies to the account.
  - **Name limit**: a 51-character name → 422 at `/categoryName`.
  - **Refused means nothing created**: a tomorrow date plus `accountName: "Brand new"` → 422 on `/date`, and no `Brand new` account exists.
  - **Idempotency**: an `Idempotency-Key` replay by name → 200 `Idempotent-Replayed`, and only one category is created.
  - **Update**: `PUT /api/v1/spendings/{id}` by name works the same way.
  - **Unchanged by id**: all 002 by-id requests behave as before · skill: backend-developer
- [X] T036 [P] [US2] Write `fe/app/lib/actions.test.ts` cases for `saveSpending(prev, formData)`:
  - **Input**: fields `mode` (`add`|`edit`), `spendingId?`, `idempotencyKey` (UUID, required for add), `date`, `merchant`, `categoryId` | `categoryName`, `accountId` | `accountName`, `amount`, `currency`.
  - **Validation**: Zod rejects with the first failing message in the data-model.md order. `amount` must match `^\d{1,8}([.,]\d{1,2})?$` and be > 0, with a comma normalised to a dot.
  - **Add**: `POST /api/v1/spendings` with header `Idempotency-Key`, body per the contract, and **exactly one** request.
  - **Edit**: `PUT /api/v1/spendings/{spendingId}`.
  - **Success**: `refresh()` is called and `{ status: 'ok', month: 'YYYY-MM' }` is returned from the saved date.
  - **API errors**: mapped through `messageFor`, with `field` set from the first `errors[].field` (`/date` → `date`).
  - **Edited item gone**: a 404 on `PUT` returns `{ status: 'error', message: 'This item no longer exists.', gone: true }` and still calls `refresh()`.
  - **No session**: `{ status: 'signed-out' }` · skill: frontend-developer
- [X] T037 [P] [US2] Write `fe/app/ui/spendings/date-picker.test.tsx`:
  - **Opening**: the field shows the date (`Oct 28, 2026`); clicking it or pressing Enter/Space opens a popover with `role="dialog"` and the label "Choose date".
  - **Grid**: an ARIA grid of the value's month, with the title `October 2026`, previous/next month buttons (labels "Previous month" / "Next month") and a "Today" button.
  - **Future days**: days after `today` are `aria-disabled` and not selectable; the next-month button is disabled when the shown month is `today`'s.
  - **Keyboard and selection**: arrow keys move focus between days; Enter picks a day, closes the popover, returns focus to the field and calls `onChange("2026-10-12")`. "Today" picks `today`.
  - **Closing**: Escape closes without changing the value.
  - **Typing**: typing `2026-10-12` or `12/10/2026` in the field and blurring sets the value; an invalid date leaves it empty · skill: frontend-developer
- [X] T038 [P] [US2] Write `fe/app/ui/spendings/combobox.test.tsx` (basic part; autofill comes in US3):
  - **Roles**: `role="combobox"` with `aria-expanded`, `aria-controls` → a `role="listbox"` of options.
  - **Picking**: clicking an option sets the value to `{ id, name }`.
  - **Typing**: typing a name that matches an option exactly (ignoring case and surrounding spaces) yields `{ id }`; a name with no match yields `{ name: <trimmed> }` and shows a hint option `Create “<name>”` at the end of the list.
  - **Label**: the visible label is wired with `aria-labelledby` · skill: frontend-developer
- [X] T039 [P] [US2] Write `fe/app/ui/spendings/entry-row.test.tsx` (add mode):
  - **Opening**: + in the table header (`aria-label="Add spending"`) opens the row at the top with `date = today` and focus in Merchant.
  - **Saving**: the check mark (`aria-label="Save"`) submits; while pending, Save is disabled and a second click sends nothing.
  - **Validation**: errors show in an `aria-live="polite"` region with the exact message, the row keeps its values, and focus moves to the failing field.
  - **Cancelling**: X (`aria-label="Cancel"`) and Escape close the row without calling the action.
  - **Navigation**: after `{ status: 'ok', month }`, if `month` differs from the shown month, `router.push('/spendings?month=' + month)`.
  - **Idempotency key**: a new one is created each time the row opens (`crypto.randomUUID`).
  - **Gone on save**: on `{ status: 'error', gone: true }` the row closes and the message shows in the table's live region (not the row's).
  - **Signed out**: when the action returns `{ status: 'signed-out' }`, the component calls `window.location.assign('/')` (stubbed in the test) · skill: frontend-developer
- [X] T040 [P] [US2] Write `fe/e2e/add-spending.spec.ts`:
  - Click +, pick a date last month in the calendar, type a merchant with the run marker, type a new category name, pick an account, type `12.50` and click Save.
  - The view switches to last month and the row, footer and card show it.
  - Settings is not needed: the new category appears in the category dropdown next time.
  - An empty merchant shows "Add a merchant name.", and a `0` amount shows "Enter an amount greater than zero."
  - Clean up afterwards · skill: frontend-developer

### Implementation for User Story 2

- [X] T041 [US2] Change `…/main/dto/spending/SpendingRequest.kt`: `categoryId: UUID?`, `categoryName: String?` (`@Size(max = 50)`), `accountId: UUID?`, `accountName: String?` (`@Size(max = 50)`), with schemas and descriptions exactly as in `specs/003-spending-screens/contracts/openapi.yaml` `SpendingRequest`. In `toCommand()`, build `CategoryRef.ById(id)` or `CategoryRef.ByName(name.trim())`; the exactly-one check throws `ValidationException(FieldViolation("/categoryId", …, "give exactly one of categoryId and categoryName"))` (and the account equivalent). Change `…/main/model/SpendingCommand.kt` to hold `category: NamedRef` and `account: NamedRef` (`sealed interface NamedRef { ById(id); ByName(name) }`) · skill: backend-developer
- [X] T042 [US2] Create `…/main/service/NamedItemResolver.kt`, reusing the name rules and `saveOrDuplicate` from `…/main/service/NamedItemErrors.kt` and the account and category repositories. Make T034 green · skill: backend-developer
- [X] T043 [US2] Change `…/main/service/SpendingService.kt`: in `create` and `update`, resolve both refs through `NamedItemResolver` in the same `@Transactional` before validating and saving. Compute the idempotency fingerprint over the **resolved ids**, so a by-name replay matches. Update `…/main/controller/SpendingController.kt` docs and examples (`byIds`, `byNames`). Adjust `…/test/service/SpendingServiceTest.kt`. Make T035 green and the 002 ITs stay green, then **regenerate types** · skill: backend-developer
- [X] T044 [US2] In `fe/app/lib/definitions.ts`, add the aliases `Category`, `Account` and `SpendingRequest`, and the UI types `NamedValue = { id: string; name: string } | { name: string }` and `Draft`. In `fe/app/lib/data.ts`, add `getCategories()` and `getAccounts()` (each one call, unpaged lists), and add both to `loadSpendingsPage`'s `Promise.all` · skill: frontend-developer
- [X] T045 [US2] Create `fe/app/lib/actions.ts` with `saveSpending`, following the shared "Server Actions" rule. Make T036 green · skill: frontend-developer
- [X] T046 [P] [US2] Create `fe/app/ui/spendings/date-picker.tsx` (`'use client'`; non-modal popover positioned under the field; ARIA grid; `today` passed in as a `YYYY-MM-DD` string, never `new Date()` for the limit). Make T037 green · skill: frontend-developer
- [X] T047 [P] [US2] Create `fe/app/ui/spendings/combobox.tsx` (`'use client'`, WAI-ARIA 1.2 combobox with a listbox popup; `options: { id, name, count }[]`; `value: NamedValue | null`; `onChange`; `label`; `name` for the hidden inputs `<name>Id` / `<name>Name`). Make T038 green · skill: frontend-developer
- [X] T048 [US2] Create `fe/app/ui/spendings/entry-row.tsx` (`'use client'`):
  - a `<tr>` containing a `<form action={formAction}>`, using `useActionState(saveSpending, …)`;
  - inputs in the order Date (date picker), Merchant (text, `maxLength=255`), Category (combobox), Account (combobox), Amount (`inputMode="decimal"`), plus the hidden `mode`, `idempotencyKey` and `currency`;
  - check and X buttons, and the live error region;
  - local validation first, using the constants from `messages.ts`.

  Wire it into `fe/app/ui/spendings/spendings-table.tsx`: a `+` button in the header, `openRow` state (`null | { mode: 'add' } | { mode: 'edit', spendingId }`), and only one open row at a time (FR-033). Make T039 and T040 green · skill: frontend-developer

**Checkpoint**: US1 and US2 work on their own (quickstart §4 rows 6–8, §2 rows 2–5).

---

## Phase 5: User Story 3 — Fast keyboard entry with autofill (Priority: P1)

**Goal**: Merchant, category and account autofill with the most used match as the user types.
Accepting a past merchant fills in its last category and account when they are empty. Tab and
Enter move between fields, and Enter saves once every field is complete.

**Independent Test**: With "Groceries" used 10 times and "Gifts" once, and "Neighbourhood
Market" recorded before: open a row, type `nei`, Tab (category and account filled), Tab, Tab,
type an amount, Enter. The spending is saved without the mouse. In a fresh row, typing `g` in
category autofills "Groceries".

### Tests for User Story 3 ⚠️ (write first, see them fail)

- [X] T049 [P] [US3] Write `…/test/service/MerchantServiceTest.kt` and `…/test/controller/MerchantsIT.kt`:
  - **Grouping and spelling**: spendings `"Loblaws"` (2026-10-01), `"loblaws "` (2026-10-20) and `"LOBLAWS"` (2026-10-20, created later) form one merchant. Its `name` is `"LOBLAWS"`, `spendingCount: 3`, `lastUsedOn: "2026-10-20"`, and `lastCategory`/`lastAccount` are those of the LOBLAWS spending.
  - **Order**: `spendingCount` desc, then `lower(name)`.
  - **Limit**: `limit=1` gives one item and `truncated: true`. `limit=0` or `limit=1001` gives 422 with pointer `limit`. The default is 500.
  - **Other cases**: another user's spendings are excluded; no spendings gives `{ items: [], truncated: false }`; 401 without a token · skill: backend-developer
- [X] T050 [P] [US3] Extend `fe/app/lib/utils.test.ts` with `rankSuggestions(typed, items: { name, count }[])`:
  - **Matching**: case-insensitive; prefix matches first, then contains-matches, each ordered by `count` desc and then name (`localeCompare`, sensitivity `base`).
  - **Empty input**: an empty `typed` returns all items by count.
  - **Completion**: `bestCompletion(typed, ranked)` returns the first **prefix** match or `null`, and returns `null` when `typed` already equals an item ignoring case and surrounding spaces, even if a longer prefix match ranks higher (spec Edge Cases) · skill: frontend-developer
- [X] T051 [P] [US3] Extend `fe/app/ui/spendings/combobox.test.tsx` (autofill):
  - **Suggestion**: typing `g` with Groceries (10) and Gifts (1) sets the input to `Groceries` with `ceries` selected (`selectionStart === 1`, `selectionEnd === 9`) and `aria-autocomplete="both"`. Typing `gi` switches to `Gifts`. Typing `gz` shows no suggestion and keeps `gz`.
  - **Accepting**: Backspace removes the selected remainder only. Tab or Enter accepts, emits `{ id, name }` and calls `onAccept` (which the row uses to move focus).
  - **Arrow keys**: ArrowDown/ArrowUp move `aria-activedescendant`, and Enter/Tab pick the active option.
  - **Escape**: closes the list first; a second Escape bubbles up.
  - **On edit**: when focus enters the field it selects the whole value, and it never autofills until a key is typed · skill: frontend-developer
- [X] T052 [P] [US3] Extend `fe/app/ui/spendings/entry-row.test.tsx` (keyboard and merchant):
  - **Merchant fill-in**: typing `nei` in Merchant autofills `Neighbourhood Market`. Tab accepts it, fills Category `Groceries` and Account `Visa` from `lastCategory`/`lastAccount` **only where those fields are empty**, and moves focus to Category with its value selected.
  - **No overwrite**: with Category already set, accepting the merchant does not change it. An unknown merchant fills nothing.
  - **Field order**: Tab moves Date → Merchant → Category → Account → Amount; Shift+Tab goes back.
  - **Enter mid-row**: with every field complete, Enter in any field submits. With a field empty, Enter moves to the next field.
  - **Last field**: Tab or Enter in Amount submits when complete; otherwise it shows the first message and focuses the first incomplete field.
  - **Escape**: with the calendar or a list open it closes that first; otherwise it cancels the row.
  - **Save count**: only one save per Enter · skill: frontend-developer
- [X] T053 [P] [US3] Write `fe/e2e/keyboard-autofill.spec.ts`:
  - Seed a merchant (marker) twice with category `Groceries` and account A.
  - Open a row and type its first 3 letters. Tab (expect the category and account filled), Tab, Tab, type `9.99`, Enter: the row is saved, **with no mouse events** after clicking +.
  - In a new row, type `g` in Category and check the autofilled text and selection.
  - Clean up afterwards · skill: frontend-developer

### Implementation for User Story 3

- [X] T054 [P] [US3] Create `…/main/model/Merchant.kt`, `…/main/dto/merchant/MerchantListResponse.kt` (`@Schema(name = "MerchantList")`: `items`, `truncated`, plus `@Schema(name = "Merchant")` with `name`, `spendingCount`, `lastUsedOn`, `lastCategory: RefDto`, `lastAccount: RefDto`) and `…/main/dto/merchant/MerchantMappings.kt` · skill: backend-developer
- [X] T055 [US3] Create `…/main/repository/spending/MerchantQueries.kt` (`JdbcClient`):
  - a CTE `latest AS (SELECT DISTINCT ON (lower(btrim(merchant))) lower(btrim(merchant)) AS k, merchant, spending_date, category_id, account_id FROM spendings WHERE user_id = :u ORDER BY lower(btrim(merchant)), spending_date DESC, created_at DESC, id DESC)`;
  - joined to `counts AS (SELECT lower(btrim(merchant)) k, count(*) n FROM spendings WHERE user_id = :u GROUP BY 1)`, and to `categories` and `accounts` for the names;
  - ordered by `n DESC, lower(merchant)`, with `LIMIT :limit + 1` to detect `truncated` · skill: backend-developer
- [X] T056 [US3] Create `…/main/service/MerchantService.kt` and `…/main/controller/MerchantController.kt`: `GET /api/v1/merchants`, operationId `listMerchants`, tag `Merchants`, `@RequestParam(defaultValue = "500") @Min(1) @Max(1000) limit: Int`, with 200 (contract example), 401, 422, 500 and 503. Make T049 green, then **regenerate types** · skill: backend-developer
- [X] T057 [US3] In `fe/app/lib/definitions.ts`, add the `Merchant` alias. In `fe/app/lib/data.ts`, add `getMerchants()` (default limit) to `loadSpendingsPage`, with an MSW-backed test in `fe/app/lib/data.test.ts`. In `fe/app/lib/utils.ts`, add `rankSuggestions` and `bestCompletion`. Make T050 green · skill: frontend-developer
- [X] T058 [US3] Add inline autofill to `fe/app/ui/spendings/combobox.tsx`: `aria-autocomplete="both"`, `setSelectionRange` for the remainder, Backspace handling, `onAccept`, select-all on focus, and Escape layering. Use the same component for Merchant in a `freeText` mode (value is a plain string; options from `getMerchants`, with `count = spendingCount`). Make T051 green · skill: frontend-developer
- [X] T059 [US3] In `fe/app/ui/spendings/entry-row.tsx`, add the FR-053–FR-058 key handling at the row level:
  - an ordered list of field refs;
  - Tab and Enter logic per T052, with the default Tab prevented only when the row decides to submit or refocus;
  - merchant acceptance filling empty category and account from the merchant's `lastCategory`/`lastAccount` as `{ id, name }`.

  No global `keydown` listeners; handlers live on the row's elements. Make T052 and T053 green · skill: frontend-developer

**Checkpoint**: Keyboard-only entry works (quickstart §4 rows 4–6; SC-001, SC-002).

---

## Phase 6: User Story 4 — Edit and delete a spending from the table (Priority: P1)

**Goal**: Hovering or focusing a row reveals edit and remove. Edit reuses the entry row with the
spending's values. Remove deletes at once, with no confirmation or undo.

**Independent Test**: Record a spending, edit its amount and category, and check the table and
API. Then remove it and check it is gone from the table, the totals and the API.

### Tests for User Story 4 ⚠️ (write first, see them fail)

- [X] T060 [P] [US4] Extend `fe/app/lib/actions.test.ts` with `deleteSpending(spendingId)`:
  - **Input**: Zod checks a UUID.
  - **Request**: exactly one `DELETE /api/v1/spendings/{id}`.
  - **Success**: `refresh()` is called and `{ status: 'ok' }` is returned.
  - **Already gone**: a 404 returns `{ status: 'error', message: 'This item no longer exists.' }` and still calls `refresh()`.
  - **Server unavailable**: the unavailable message, with no refresh · skill: frontend-developer
- [X] T061 [P] [US4] Extend `fe/app/ui/spendings/spendings-table.test.tsx` (actions):
  - **Visibility**: the edit (`aria-label="Edit <merchant> on <date>"`) and remove (`aria-label="Remove <merchant> on <date>"`) buttons are visually hidden until the row is hovered **or** contains focus (`group-hover`/`group-focus-within`), and always present in the accessibility tree and Tab order.
  - **Edit**: opens the entry row in place, with the date, merchant, category `{id,name}`, account `{id,name}` and amount (as typed text, `"84.37"`) prefilled; it does not create an idempotency key.
  - **Cancel**: shows the original row.
  - **Remove**: calls `deleteSpending` once and disables the button while pending. On error the row stays and the message shows in the table's live region.
  - **One open row**: while a row is open, + and every edit button are disabled, and remove on the open row is hidden (FR-062).
  - **Signed out**: when `deleteSpending` returns `{ status: 'signed-out' }`, the table calls `window.location.assign('/')` (stubbed in the test) · skill: frontend-developer
- [X] T062 [P] [US4] Write `fe/e2e/edit-delete.spec.ts`:
  - Seed a spending (marker), hover its row, edit the amount and category, and save. The table and footer update. Move its date to last month: the view switches.
  - Focus a row with Tab and check the buttons are reachable.
  - Click remove: it disappears immediately, the footer count drops, and the month line total changes.
  - Clean up afterwards · skill: frontend-developer

### Implementation for User Story 4

- [X] T063 [US4] Add `deleteSpending` to `fe/app/lib/actions.ts`. Make T060 green · skill: frontend-developer
- [X] T064 [US4] In `fe/app/ui/spendings/spending-row.tsx` and `fe/app/ui/spendings/spendings-table.tsx`, add the row actions (inline SVG pencil and trash icons, `aria-hidden` icons inside labelled `<button>`s, at least 44 px targets), edit mode through `entry-row.tsx` in `mode: 'edit'` with the hidden `spendingId`, delete through `useTransition`, and the table-level live region. Make T061 and T062 green · skill: frontend-developer

**Checkpoint**: All P1 stories work. **This is the first release worth deploying** (see
Implementation Strategy).

---

## Phase 7: User Story 5 — See where the month's money went (Priority: P2)

**Goal**: The By category card shows every category with spending in the month, and only those,
as a table (amount, share, total row) or as a donut with one slice per category and a legend.
There is no cap and no "Everything else".

**Independent Test**: With October spendings in eight categories, the table lists those eight
with amount, share and total. Pie shows eight slices and a legend. The toggle is kept across
month changes.

### Tests for User Story 5 ⚠️ (write first, see them fail)

- [X] T065 [P] [US5] Write `…/test/controller/CategoryTotalsIT.kt` and extend `…/test/service/TotalsServiceTest.kt` for `byCategory(identity, from, to)`:
  - **Contents**: only categories with spending in range, ordered by total desc then `lower(name)`; each `total` an exact decimal and `spendingCount ≥ 1`; the envelope `total` and `spendingCount` equal the sums of the items. A range with nothing gives `items: []`, `total: "0.00"`, `spendingCount: 0`.
  - **Validation**: `from` after `to` gives 422 at `from`; more than 366 days gives 422 at `to`; a missing `from` or `to` gives 422.
  - **Other cases**: ownership is respected, and 401 without a token · skill: backend-developer
- [X] T066 [P] [US5] Extend `fe/app/lib/utils.test.ts` with `donutArcs(items: { cents }[], { r, inner })`. Angles come from integer cents, start at −90°, and sum to 360°. A single item gives a full ring (two arcs, or a circle path). Zero items gives `[]`. Each arc's `largeArc` flag is correct for shares over 50% · skill: frontend-developer
- [X] T067 [P] [US5] Write `fe/app/ui/spendings/by-category-card.test.tsx`:
  - **Toggle**: a two-button group "Table" / "Pie" with `aria-pressed`, `table` by default. The choice is saved to `localStorage` key `hlb.byCategoryView` inside `try/catch` (a throwing storage still works) and restored after remount.
  - **Table view**: one row per item with the name, `formatMoney(total)` and `sharePercent` + `%`, largest first, plus a "Total" row equal to the month total.
  - **Pie view**: the SVG has `role="img"` and an `aria-label` summarising the top category. There is one `<path>` per item (twelve items → twelve paths) and **no** "Everything else" text. Each slice has a `<title>` with "<name>: <amount> (<share>%)" and is focusable. The legend is an ordered list with a swatch, name, amount and share.
  - **Colours**: adjacent slices never share a colour (12-colour palette, R-010).
  - **Empty month**: no items shows "Nothing spent in October 2026." · skill: frontend-developer
- [X] T068 [P] [US5] Write `fe/e2e/by-category.spec.ts`:
  - Seed three spendings in two categories (marker).
  - The table shows exactly two category rows plus Total.
  - Switching to Pie shows two slices and a legend.
  - Going to the previous month keeps the Pie view, and an empty month shows the empty message.
  - Clean up afterwards · skill: frontend-developer

### Implementation for User Story 5

- [X] T069 [P] [US5] Create `…/main/model/CategoryTotal.kt`, `…/main/dto/totals/CategoryTotalsResponse.kt` (`@Schema(name = "CategoryTotals")` with `from`, `to`, `total`, `spendingCount`, `items`; `@Schema(name = "CategoryTotal")` with `category: RefDto`, `total`, `spendingCount`), and the mappings in `…/main/dto/totals/TotalsMappings.kt` · skill: backend-developer
- [X] T070 [US5] Add `byCategory(userId, from, to)` to `…/main/repository/spending/TotalsQueries.kt`: `SELECT c.id, c.name, sum(s.amount), count(*) FROM spendings s JOIN categories c ON c.id = s.category_id AND c.user_id = s.user_id WHERE s.user_id = :u AND s.spending_date BETWEEN :from AND :to GROUP BY c.id, c.name ORDER BY 3 DESC, lower(c.name)`. Add `TotalsService.byCategory`, which validates the range (`from ≤ to`, at most 366 days) with `ValidationException`. Add `GET /api/v1/category-totals` (operationId `listCategoryTotals`, required ISO `from` and `to`) to `…/main/controller/TotalsController.kt`. Make T065 green, then **regenerate types** · skill: backend-developer
- [X] T071 [US5] Add the `CategoryTotals` alias and `getCategoryTotals({ from, to })` (with a `data.test.ts` case) and include it in `loadSpendingsPage`. Add `donutArcs` to `fe/app/lib/utils.ts`. Make T066 green · skill: frontend-developer
- [X] T072 [US5] Load the `dataviz` skill for the palette and labelling. Create:
  - `fe/app/ui/spendings/category-table.tsx` (server-renderable);
  - `fe/app/ui/spendings/category-pie.tsx` (SVG donut, `r = 84`, `inner = 54` as in the design, with a 12-colour palette that passes 3:1 against white and differs in lightness between neighbours; define it in `globals.css` as `--color-cat-1`…`--color-cat-12`);
  - `fe/app/ui/spendings/by-category-card.tsx` (`'use client'`, toggle only).

  Place the card in `fe/app/ui/spendings/month-section.tsx` under the total card. Make T067 and T068 green · skill: dataviz, frontend-developer

**Checkpoint**: US5 works on its own (quickstart §4 row 11).

---

## Phase 8: User Story 6 — Manage categories and accounts in Settings (Priority: P2)

**Goal**: The user icon opens a Settings dialog with Categories and Accounts sections. Users can
add and remove their own categories (defaults are locked) and accounts, with the spec's
duplicate, in-use and last-account messages.

**Independent Test**: Open Settings, add the category "Coffee" and the account "Cash", check
they appear in Settings and in the entry row's dropdowns, then remove both. The duplicate,
in-use and last-account cases show their messages.

### Tests for User Story 6 ⚠️ (write first, see them fail)

- [X] T073 [P] [US6] Write `…/test/controller/LastAccountIT.kt` and extend `…/test/service/AccountServiceTest.kt`:
  - **The rule**: with exactly one unused account, `DELETE /api/v1/accounts/{id}` gives 422 `urn:problem:hl-budget:unprocessable-entity` with `details.reason: "last-account"`, and the account still exists. With two unused accounts, deleting one gives 204 and deleting the other gives 422.
  - **Check order**: a used account (even the last one) gives 409 `in-use` first; another user's account gives 404 first.
  - **Race**: two concurrent deletes of a user's last two unused accounts leave exactly one account (one 204, one 422) · skill: backend-developer
- [X] T074 [P] [US6] Extend `fe/app/lib/actions.test.ts` with `createCategory`, `deleteCategory`, `createAccount` and `deleteAccount`:
  - **Input**: Zod trims names and requires 1–50 characters (empty gives `Type a name for the category.` / `Type a name for the account.`).
  - **Requests**: exactly one `POST /api/v1/categories|accounts` with `{ name }`, or one `DELETE …/{id}`.
  - **Errors**: 409 `duplicate-name` → `“<name>” already exists.`; 409 `in-use` → the category or account wording with the count; 422 `last-account` → `You need at least one account.`; 422 `default-category` → `Default categories can’t be removed.`
  - **Success**: `refresh()` is called · skill: frontend-developer
- [X] T075 [P] [US6] Write `fe/app/ui/settings/settings-dialog.test.tsx`:
  - **Opening**: the header's Settings button opens a `<dialog>` via `showModal` (polyfill `HTMLDialogElement.prototype.showModal` in `fe/vitest.setup.ts` if jsdom lacks it). Its title is "Settings". A vertical `role="tablist"` (`aria-orientation="vertical"`) has the tabs "Categories" and "Accounts"; arrow keys move between them.
  - **Closing**: the close button (`aria-label="Close settings"`), Escape and a click on the backdrop each close it, and focus returns to the Settings button.
  - **Categories panel**: the heading "Default categories · 17" lists the defaults, each with a lock icon (`aria-label="Default category"`), its "N spendings recorded" / "No spendings yet" text, and no remove button. "Your categories" lists the custom ones, each with "N spendings recorded" / "No spendings yet" and a remove button labelled "Remove <name>". An add field (label "New category") and an "Add" button.
  - **Local checks**: an empty add shows "Type a name for the category."; a case-insensitive duplicate shows "“coffee” already exists." without calling the action. Server messages show in an `aria-live` region.
  - **Accounts panel**: the same pattern, with "Type a name for the account." and the in-use and last-account messages.
  - **Signed out**: when any Settings action returns `{ status: 'signed-out' }`, the dialog calls `window.location.assign('/')` (stubbed in the test).

  Split the panel cases into `fe/app/ui/settings/categories-panel.test.tsx` and `fe/app/ui/settings/accounts-panel.test.tsx` if clearer · skill: frontend-developer
- [X] T076 [P] [US6] Write `fe/e2e/settings.spec.ts`:
  - Open Settings from the user icon. Add a category (marker) and check it appears in the entry row's category dropdown. Add `<marker>` again in upper case and see the duplicate message.
  - Use the category on a spending and try to remove it: the in-use message.
  - Remove the spending, then remove the category.
  - Accounts: add one and remove it.
  - Escape closes the dialog.
  - Clean up afterwards · skill: frontend-developer

### Implementation for User Story 6

- [X] T077 [US6] Implement the last-account rule (R-006):
  - In `…/main/repository/account/AccountRepository.kt`, add `@Lock(PESSIMISTIC_WRITE) findAllByUserId(userId)` (or a native `SELECT … FOR UPDATE`) and `countByUserId(userId)`.
  - In `…/main/service/AccountService.kt`'s `delete`, take the lock on all the caller's accounts, then check not found → `requireUnused` → `if (count == 1) throw lastAccount(id)`.
  - In `…/main/service/NamedItemErrors.kt`, add `lastAccount(id) = UnprocessableEntityException("Account $id is the only account", mapOf("reason" to "last-account"))`.
  - Document the 422 (with the contract example) on `deleteAccount` in `…/main/controller/AccountController.kt`.

  Make T073 green, then **regenerate types** · skill: backend-developer
- [X] T078 [US6] Add the four actions to `fe/app/lib/actions.ts`, extending `messageFor` for `default-category`. Make T074 green · skill: frontend-developer
- [X] T079 [US6] Create:
  - `fe/app/ui/settings/settings-dialog.tsx` (`'use client'`; its own trigger button rendered into `AppHeader`'s `settingsSlot` from `fe/app/spendings/page.tsx`, with an inline SVG user icon);
  - `fe/app/ui/settings/categories-panel.tsx`;
  - `fe/app/ui/settings/accounts-panel.tsx`.

  The categories and accounts come from a small async server wrapper inside `<Suspense fallback={null}>` that calls `getCategories()` and `getAccounts()` and passes plain arrays to the dialog. Make T075 and T076 green · skill: frontend-developer

**Checkpoint**: US6 works on its own (quickstart §4 rows 12–14).

---

## Phase 9: User Story 7 — Sign in and out (Priority: P3)

**Goal**: The header shows Sign in when signed out, and Sign out plus the user icon when signed
in. Sign-in lands on `/spendings`, and sign-out returns to the signed-out screen with no data.

**Independent Test**: From `/`, sign in and land on `/spendings` for the current month. Sign
out: the signed-out screen shows, and Back does not reveal spendings.

### Tests for User Story 7 ⚠️ (write first, see them fail)

- [X] T080 [P] [US7] Update `fe/e2e/sign-in.spec.ts`, `fe/e2e/sign-out.spec.ts` and `fe/e2e/protected-routes.spec.ts`:
  - Sign in from `/` → `/spendings` with today's month.
  - Sign out → `/` shows "You're signed out", and `page.goBack()` shows no spending text.
  - `/spendings` and `/home` without a session → the Auth0 login page, with `returnTo` set to `/spendings`.

  Remove the 001 profile-card assertions · skill: frontend-developer

### Implementation for User Story 7

- [X] T081 [US7] Make sure the Sign in link on `/` and in the header carries `returnTo=/spendings`, and that the Auth0 `signInReturnTo` default (in `fe/app/lib/auth0.ts`, if it exists) is `/spendings` rather than `/home`. Make sure `fe/app/lib/safe-return-to.ts` accepts `/spendings?month=YYYY-MM`, extending `fe/app/lib/safe-return-to.test.ts` to match. Make T080 green · skill: frontend-developer

**Checkpoint**: All user stories are independently functional.

---

## Phase 10: Polish & Cross-Cutting Concerns

**Purpose**: Performance and accessibility proof, docs, gates and release.

- [X] T082 [P] Write `…/test/controller/TotalsPerformanceIT.kt`. Seed 10,000 spendings for one user across 3 years, 15 categories, 3 accounts and about 400 merchant spellings, using the batch insert helper from 002's `SpendingListPerformanceIT`. Assert that `monthly-totals`, `category-totals` (one month), `merchants` and one month of `GET /spendings` each answer in **< 200 ms** (median of 5, after a warm-up), serving SC-005 · skill: backend-developer
- [X] T083 [P] Write `fe/e2e/a11y.spec.ts` with `@axe-core/playwright`. Scan `/` signed out, and `/spendings` in each of these states: plain, with the entry row open, with the calendar open, with a combobox list open, with the Pie view, and with Settings open on each tab. Assert no `serious` or `critical` violations (SC-008) · skill: frontend-developer
- [X] T084 [P] Update `hl-budget-backend/README.md` (the new endpoints and the by-name spending fields) and `hl-budget-frontend/README.md` (routes `/` and `/spendings`, the new e2e specs and their data-cleanup marker, the axe check) · skill: backend-developer, frontend-developer
- [ ] T085 Run the full gates. Backend: `./gradlew check`; confirm `git diff origin/main -- contracts/api.yaml` contains only what `specs/003-spending-screens/contracts/openapi.yaml` describes. Frontend: `pnpm typecheck && pnpm lint && pnpm test && pnpm build && pnpm e2e`. Then run quickstart §2 and §4 by hand · skill: backend-developer, frontend-developer
- [ ] T086 Operator step (ask the user first): open the backend PR, merge it after review, and let the release workflow open the hl-helm image PR; merge that. Then do the same for the frontend. Backend first: its additions are inert for the running frontend · skill: none
- [ ] T087 Verify both rollouts with `homelab:hl-deploy-status` (Argo CD Synced/Healthy, pods Ready). Sign in on the live host and repeat quickstart §4 rows 2, 6 and 11. Record the result in the **Status** line of `specs/003-spending-screens/spec.md` · skill: homelab:hl-deploy-status

---

## Phase 11: Post-implementation review fixes (2026-10-28)

**Purpose**: Fixes from the user's hands-on review of the running screens. Each is recorded as a
clarification in spec.md (Clarifications, post-implementation review) and in the FRs named below.
Frontend only; no API or contract change. All paths are in `hl-budget-frontend/`.

- [X] T088 [US2] The calendar was cut off by the table card (`overflow-hidden` / `overflow-x-auto`). Render it in a portal on `document.body` with fixed position, under the field or above it when there is no room, inside the window, following scroll and resize, in `app/ui/spendings/date-picker.tsx` using the new `app/ui/spendings/use-floating.ts` (research R-010 revision). Serves FR-034 · Evidence: `date-picker.test.tsx` › "opens outside any clipping container, and a press outside closes it" · skill: frontend-developer
- [X] T089 [US3] The merchant, category and account suggestion lists had the same clipping. Portal them the same way, at least as wide as their field; `aria-controls` is set only while the list is open, in `app/ui/spendings/combobox.tsx`. Serves FR-051 · Evidence: `combobox.test.tsx` › "opens its list outside any clipping container" · skill: frontend-developer
- [X] T090 [US1] The merchant column was about 50 px wide: the fixed columns took 672 of the table's 740 px. Widen the page (`max-w-6xl` → `max-w-7xl` in `app/ui/app-header.tsx`, `app/ui/skeletons.tsx`, `app/ui/spendings/month-section.tsx`, `app/spendings/error.tsx`), put the summary cards beside the table only from `xl` (1280 px), narrow the amount column to `w-28`, and raise the table's `min-w` to 820 px, so the merchant column stays at least about 160 px wide (about 210 px beside the cards, 300 px or more below `xl`), in `app/ui/spendings/spendings-table.tsx` and `entry-row.tsx`. Serves FR-024 · Evidence: column arithmetic above; no automated layout test (checked by the user in the browser) · skill: frontend-developer
- [X] T091 [US2] A new row opens its calendar by itself (`autoOpen`), and picking a day moves focus to Merchant (`onPicked`); editing does not open it. Escape on the date field, with the calendar closed, cancels the row (it did not before). Changes are in `app/ui/spendings/date-picker.tsx` and `entry-row.tsx`. Update the e2e helpers for the open calendar: new `openEntryRow` in `e2e/support.ts`, used by `add-spending.spec.ts` and `keyboard-autofill.spec.ts`; `addSpendingViaUi` picks from the open calendar; `a11y.spec.ts` scans the row with its calendar open. Serves FR-030, FR-056 · Evidence: `date-picker.test.tsx` › "can open by itself, and reports a pick but not a close"; `entry-row.test.tsx` › "a new row opens its calendar, and picking a day moves on to Merchant", "editing does not open the calendar", "Escape on the date field cancels the row once the calendar is closed" · skill: frontend-developer
- [X] T092 [US3] Left and Right arrows move between the row's fields once the cursor is at that end of the text, or the whole text is selected; a field reached this way has its text selected. Modifier keys keep the browser's behaviour, and the calendar's own arrow keys are unaffected. Implemented by `arrowStep` and `arrowMove` in `app/ui/spendings/entry-row.tsx`, plus an `onKeyDown` prop for the date field in `date-picker.tsx`. Serves FR-053 · Evidence: `entry-row.test.tsx` › "Left and Right arrows move between fields at the ends of the text" · skill: frontend-developer
- [X] T093 [US2] A new spending starts in the shown month: today in the current month, otherwise the 1st of the shown month, via `defaultDateFor` in `app/lib/utils.ts`, used by `spendings-table.tsx`. Serves FR-030 · Evidence: `utils.test.ts` › "a new spending starts today in the current month, else on the 1st of the shown month" · skill: frontend-developer
- [X] T094 [US2] Enter on the Spendings screen opens a new row when nothing has focus, no row is open and no dialog is open; Enter on a focused control keeps its own meaning. The + button gets `aria-keyshortcuts="Enter"` and the title "Add a spending row (Enter)". Changes are in `app/ui/spendings/spendings-table.tsx`. Serves FR-030 · Evidence: `spendings-table.test.tsx` › "Enter with nothing focused opens a new row, but not while something has focus", "Enter on a focused control keeps its own meaning" · skill: frontend-developer

- [X] T095 [US3] First signed-in `pnpm e2e` run (by the user, 2026-10-28): 24 passed, 1 skipped, 3 failed. (a) App bug: accepting a past merchant fills category and account, but when focus reached the field before the filled value, the cursor sat after it and typed text was appended ("GroceriesGroceries" was saved as a new category). Fix: `app/ui/spendings/combobox.tsx` selects a value set from outside while the field has focus. Serves FR-058 · Evidence: `combobox.test.tsx` › "a value filled from outside while focused is selected, so typing replaces it", which fails without the fix and passes with it. (b) Test bug: `e2e/settings.spec.ts` matched the "Add" button loosely, and the marker `e2e-add` also matched "Remove Cat e2e-add-…"; it now uses `exact: true`. (c) Flaky login: on Auth0's page the password was typed into the email field; `completeUniversalLogin` in `e2e/support.ts` now fills and checks both fields, retrying until both hold. Commit 580e74e · skill: frontend-developer

**Gate evidence (2026-10-28, `hl-budget-frontend` branch `003-spending-screens`, commit
b653935)**:

| Gate | Result |
|------|--------|
| `pnpm typecheck` | pass |
| `pnpm lint` | pass (0 problems) |
| `pnpm test` | 24 files, 185 tests passed (175 before this phase; 10 new); 186 after T095 |
| `pnpm build` | compiled successfully |
| `pnpm e2e` (no credentials) | 7 passed, 21 skipped (signed-in journeys need `E2E_USERNAME`/`E2E_PASSWORD`) |
| `pnpm e2e` (signed in, run by the user) | 24 passed, 1 skipped, 3 failed → fixed in T095 |
| `pnpm e2e` (signed in, rerun by the user) | 25 passed, 1 skipped, 2 failed, both test bugs: a loose "Groceries" match also hit the leftover "GroceriesGroceries" category, and `addSpendingViaUi` expected one row per merchant though the autofill spec adds the same merchant twice. Fixed in 650e368 (exact match; `.first()`); the app fix from T095 held. Third run pending (T085) |
| Browser check by the user | calendar, lists, merchant width, auto-open calendar, arrows, default date and Enter confirmed ("All good") |

**Checkpoint**: Review fixes done; T085 needs a clean signed-in e2e rerun after T095; T086–T087 remain open.

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: T001 blocks everything. T002 can run after T001.
- **Foundational (Phase 2)**: depends on Setup and blocks every story.
  - The backend pieces T003 and T004 are independent of the frontend ones.
  - Frontend: tests T005–T010 first, then T011 (needs T005), T012, T013, T014 in parallel, then
    T015 → T016. T017, T018 and T019 can run in parallel.
- **US1 (Phase 3)**: after Phase 2. Backend T026 → T027 → T028 → T029 (regenerate) → frontend
  T030 → T031/T032 → T033.
- **US2 (Phase 4)**: after US1 (it builds on the table and `loadSpendingsPage`). Backend
  T041 → T042 → T043 (regenerate) → T044 → T045, with T046/T047 in parallel → T048.
- **US3 (Phase 5)**: after US2 (it extends the combobox and the entry row). Backend
  T054 → T055 → T056 can run in parallel with US2's frontend work.
- **US4 (Phase 6)**: after US2 (it reuses the entry row). It is independent of US3.
- **US5 (Phase 7)**: after US1 only. The backend part T069–T070 can start right after Phase 2.
- **US6 (Phase 8)**: after US1 (the header and page). The backend T077 can start right after
  Phase 2. The e2e T076 uses the entry row (US2).
- **US7 (Phase 9)**: after Phase 2. The e2e checks need US1's page.
- **Polish (Phase 10)**: after all stories. T082 needs US1, US3 and US5's backend parts. T086
  comes after T085, and T087 after T086.

### User Story Dependencies

```text
Phase 2 ──┬── US1 ──┬── US2 ──┬── US3
          │         │         └── US4
          │         ├── US5
          │         └── US6 (e2e also needs US2)
          └── US7 (e2e needs US1)
```

### Within Each User Story

- The backend comes first (its tests, then models and DTOs, then queries, service and
  controller), then **regenerate types**, then the frontend (tests first, then data, actions
  and components).
- Each story is done when its unit, component and e2e tests are green.

### Parallel Opportunities

- **Backend vs frontend**: every story's backend part (T026–T029, T041–T043, T054–T056,
  T069–T070, T077) depends only on Phase 2. One session can build all the backend changes while
  another builds US1's screens, as long as each type regeneration is pulled in before the
  frontend task that needs it.
- **Phase 2 tests**: T005–T010 touch different files.
- **Within stories**: all `[P]` test tasks together; T031/T032 (US1); T046/T047 (US2).
- **After US2**: US3 and US4 can proceed in parallel, as can US5 and US6 after US1.
- **Polish**: T082, T083 and T084 in parallel.

---

## Parallel Example: User Story 2

```text
# Tests first, all different files:
T034 NamedItemResolverTest.kt
T035 SpendingByNameIT.kt
T036 actions.test.ts (saveSpending)
T037 date-picker.test.tsx
T038 combobox.test.tsx
T039 entry-row.test.tsx
T040 e2e/add-spending.spec.ts

# Backend in order: T041 SpendingRequest → T042 NamedItemResolver → T043 SpendingService (regenerate types)
# Frontend: T044 data → T045 saveSpending, with T046 date-picker and T047 combobox in parallel → T048 entry-row
```

## Parallel Example: backend track alongside US1

```text
Session A (backend): T041–T043 → T054–T056 → T069–T070 → T077   (regenerate after each)
Session B (frontend): T030 → T031/T032 → T033 → (pull types) T044 …
```

---

## Implementation Strategy

### MVP First (User Story 1)

1. Phase 1, then Phase 2: the shell, routes and helpers are green.
2. Phase 3 (US1): a signed-in user sees their months and spendings. **Stop and validate** with
   T025 and quickstart §4 rows 1–3.

US1 alone is deployable but read-only. The first release worth deploying is **US1 + US2 + US3 +
US4** (all P1): view, record (fast, by keyboard), edit and delete spendings.

### Incremental Delivery

1. Foundation → US1 → validate (month view).
2. US2 → validate (add; SC-003).
3. US3 → validate (keyboard and autofill; SC-001, SC-002).
4. US4 → validate (edit and delete). **Release candidate**: T082–T087.
5. US5 → validate (By category). It can ship in the same release or right after.
6. US6 → validate (Settings).
7. US7 → validate (sign in/out journeys), then the final release.

---

## Notes

- `[P]` means different files with no dependency on an unfinished task.
- Before changing behaviour, a test for it fails first.
- No migration in this feature. If one turns out to be needed, it is a new `V4`; never edit
  V1–V3.
- No money arithmetic outside `toCents`/`centsToAmount`/`sharePercent`/`donutArcs` on the
  frontend, and only `BigDecimal`/`numeric` on the backend.
- No new runtime dependency on either side. Ask before adding one.
- Never pass the session, a token or a class instance to a client component.

---

## Implementation notes (2026-10-28)

What was built differs from the task text in these places; the code and tests follow the notes.

- **T001**: both service repos are on `003-spending-screens`. The spec repo's
  `specs/003-spending-screens/` is committed on its own `003-spending-screens` branch, cut from
  `main`, as for features 001 and 002.
- **T021, T065**: the monthly and category totals ITs are one file,
  `…/test/controller/TotalsIT.kt`.
- **T034–T043**: a spending's category and account by name go through `NamedItemResolver` (service)
  and `NamedItemInserts` (repository, `INSERT … ON CONFLICT (user_id, lower(name)) DO NOTHING`),
  so a concurrent save naming the same new item cannot abort the transaction. Error pointers use
  the existing `#/categoryId` form.
- **T037, T046**: following the design, the date field is a button that opens the calendar; typing
  a date is not supported. The calendar is a `<table>` of day buttons rather than an ARIA grid.
- **T045, T048**: the entry row calls `saveSpending` with a plain object rather than `FormData`
  (no `<form>` inside the table row); the no-JavaScript fallback is dropped, as recorded in
  plan.md Complexity Tracking.
- **T051**: autofill keeps the letters as the user typed them (`g` → `groceries`, rest selected)
  and settles on the item's own spelling when accepted, so Backspace never changes their case.
- **Shared rule 6**: a lost session sends the user to `/` with `router.replace('/')` (Next.js lint
  forbids `window.location.assign` for internal pages).
- **T009, T016**: `/` stays public and in the sitemap, as in feature 001 (it shows no personal
  data); `/spendings` is `noindex` and disallowed in robots.txt.
- **T033, T079**: the header shows "HL-Budget" and the user button is labelled "User settings",
  as in the design.
- **T082**: the performance IT is `…/test/controller/TotalsPerformanceIT.kt`; all four reads ran
  under 200 ms over 10,000 spendings.
- **Phase 11 (T088–T094)**: added after the user's hands-on review; see that phase for the
  changes, the FRs they serve and the gate evidence.
- **T085 (open)**: `./gradlew check` and `pnpm typecheck && pnpm lint && pnpm test && pnpm build`
  are green. `pnpm e2e` ran 7 signed-out checks (all passed); the 21 signed-in journeys and the
  signed-in axe scans need `E2E_USERNAME`/`E2E_PASSWORD` and a running backend. Quickstart §2 and
  §4 by hand are not done yet.

