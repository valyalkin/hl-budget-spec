# Implementation Plan: Spending Screens (Web App)

**Branch**: `003-spending-screens` | **Date**: 2026-10-28 | **Spec**: [spec.md](./spec.md)

**Input**: Feature specification from `/specs/003-spending-screens/spec.md`

## Summary

This feature builds HL Budget's first real screens in `hl-budget-frontend`, following the design
canvas: a Spendings screen with a year and month line, an inline-editable month table, a total
card and a By category card (Table / Pie), plus a Settings dialog for categories and accounts.
It also adds small backend changes in `hl-budget-backend`.

**Backend** (all additive to v1, R-002, R-004, R-005, R-006):
- `GET /monthly-totals` and `GET /category-totals`: aggregate reads for the month line and the
  By category card, so the frontend never sums money.
- `GET /merchants`: past merchants with their use count and latest category and account, for
  merchant autofill.
- `SpendingRequest` accepts `categoryName` / `accountName` as an alternative to the ids. The
  backend reuses a case-insensitive match or creates the item in the same transaction as the
  spending.
- Deleting the caller's only account is refused (422 `last-account`).

**Frontend** (R-001, R-009 – R-012):
- `/spendings?month=YYYY-MM`, protected by the proxy; `/` becomes the signed-out screen; `/home`
  redirects.
- Server Components fetch and render; small client components own the interaction: the table
  and entry row with Tab/Enter rules, an ARIA combobox with inline autofill, a date picker, the
  SVG donut and the `<dialog>` Settings.
- Writes are Server Actions that call the API and `refresh()` the page.

No new tables, environment variables or runtime dependencies. One new dev dependency:
`@axe-core/playwright`.

## Technical Context

**Language/Version**:
- Frontend: TypeScript 5.9 (strict), Node 22, React 19.
- Backend: Kotlin 2.3 on Java 21.

Both are unchanged from 001 and 002.

**Primary Dependencies**:
- Frontend: Next.js 16 (App Router, Cache Components, React Compiler), Tailwind 4,
  `@auth0/nextjs-auth0` 4, Zod 4, `openapi-typescript`. **No new runtime dependency** (R-010):
  combobox, calendar and donut are built in-house. New dev dependency: `@axe-core/playwright`
  (SC-008).
- Backend: Spring Boot 4.1, springdoc 3.1, as in 002. No new dependency.

**Storage**: PostgreSQL 17, unchanged. No migration: the new reads are `GROUP BY` queries over
the existing `ix_spendings_user_date` index (data-model.md).

**Testing**:
- Frontend: Vitest + React Testing Library + user-event + MSW for units, data and actions;
  Playwright for one spec per user story plus axe scans.
- Backend: JUnit 5, MockK, Testcontainers Postgres ITs, with Kover gates 80/75 overall and 75
  per package (R-013).

**Target Platform**: Modern evergreen desktop browsers, usable at tablet and phone widths. Both
services run on the homelab k3s cluster via `charts/apps/hl-budget`.

**Project Type**: A two-service web application (frontend + backend API).

**Performance Goals**:
- Switching months shows the new month in under 1 s at 10k spendings (SC-005). Each of the
  four reads takes under 200 ms in an IT.
- After a write, the screen is updated in under 1 s (SC-003).
- Autofill reacts on every keystroke with no network call (R-005).

**Constraints**:
- No float money: integer cents in the UI, R-008.
- Tokens stay server-side; per-user data is never cached.
- Contract changes are additive v1 changes only.
- CSP is unchanged: fonts are self-hosted with `next/font`.
- Keyboard-only operation and WCAG AA contrast (FR-091, SC-008).

**Scale/Scope**:
- A handful of users, each with up to about 10k spendings, tens of categories and accounts, and
  hundreds of merchants.
- Frontend: 2 routes, about 15 components, 6 Server Actions, 6 data functions.
- Backend: 3 read operations and 3 changed ones.

All unknowns are resolved in [research.md](./research.md) (R-001 … R-014). No
`NEEDS CLARIFICATION` remains.

## Skills that drive the implementation

| Skill | Owns | Used in |
|---|---|---|
| `backend-developer` | Totals and merchants (controller, service, repository queries, DTOs, full `@ApiResponse` sets), id-or-name resolution in `SpendingService`, last-account rule, tests, `contracts/api.yaml` export, `./gradlew check` | Phase A |
| `frontend-developer` | Routes, proxy, data layer, Server Actions, components, tests, e2e, pnpm gates | Phase B |
| `dataviz` | Donut and legend: palette, contrast, accessible labelling | US5 |
| `homelab:hl-deploy-status` | Verify both rollouts | Deploy |

Where a skill and the constitution disagree, the constitution wins. This plan has one
reconciliation, the forms-without-JavaScript rule (see Complexity Tracking).

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

| Principle | Requirement | Plan | Status |
|---|---|---|---|
| I. Spec-driven, foundation-first | Spec → plan → tasks; independently deployable increment | Spec clarified (4 answers). Backend additions deploy first and are inert for the current frontend; then the frontend deploys | PASS |
| II. Separate services, contract-first | Backend owns data and rules; frontend has no business rules and is not a BFF; generated types; RFC 9457 errors | Name resolution, last-account rule and all money sums are in the backend (R-002, R-004, R-006). Each Server Action makes exactly one API call. Types come from the regenerated `contracts/api.yaml`. The UI reads `details.reason` from problems (R-007) | PASS |
| III. Secure by default | Session in an HttpOnly cookie; no tokens in the browser; per-user scoping; authed routes noindex | `/spendings` is gated in `proxy.ts`. Actions re-check the session. Client components get only serialisable data, never the session or token. New backend reads filter on the caller's `user_id`, with ownership ITs. `/` and `/spendings` are `noindex` | PASS |
| IV. Tests and quality gates | Tests ship with code; Playwright per new flow; gates green | R-013: units for every util, data function, action and component; one e2e per story plus axe; backend ITs and a performance IT; `pnpm typecheck && pnpm lint && pnpm test` and `./gradlew check` | PASS |
| V. Homelab-ready operations | Container contract, env-only config, Flyway immutable, deploy via hl-helm | No env, secret, chart or migration change. Releases follow the existing workflows | PASS |
| VI. Simplicity at personal scale | No new services or caches; justify additions | No new runtime dependency or service. Merchants are derived, not stored. The only added dependency is a dev tool for SC-008 | PASS |
| VII. Versioned, consistent API | `/api/v1`, plural kebab-case resources, precise codes, Money as decimal string, paged collections, documented with examples | `/api/v1/monthly-totals`, `/category-totals`, `/merchants` (GET, 200/401/422/500/503); `MoneyTotal` is a decimal string; 422 `last-account`; request fields relaxed from required to optional (allowed). The new collections are bounded and unpaged; `merchants` is capped by `limit` (Complexity Tracking) | PASS (justified deviation) |
| Tech & platform constraints | Listed stack; no float money | Stack unchanged; UI money maths in integer cents; sums in SQL `numeric` | PASS |

**Contract compatibility statement**: **backward compatible**, v1.
- Added: 3 GET operations and the `MoneyTotal`, `MonthTotal`, `MonthlyTotals`, `CategoryTotal`,
  `CategoryTotals`, `Merchant` and `MerchantList` schemas.
- Changed compatibly:
  - `SpendingRequest.categoryId` and `accountId` go from required to optional, and the optional
    `categoryName` and `accountName` are added. A request valid in 002 is still valid and means
    the same.
  - `DELETE /accounts/{id}` gains a documented 422 reason.
- Removed or renamed: nothing.

**Post-design re-check (after Phase 1)**: PASS. [data-model.md](./data-model.md) adds no
storage. [contracts/openapi.yaml](./contracts/openapi.yaml) holds only the additions above. The
two deviations are recorded below.

## Project Structure

### Documentation (this feature)

```text
specs/003-spending-screens/
├── plan.md              # This file
├── research.md          # Phase 0: decisions R-001 … R-014
├── data-model.md        # Phase 1: derived read models, write rules, UI state
├── quickstart.md        # Phase 1: gates, backend checks, browser journey, e2e, deploy
├── contracts/
│   └── openapi.yaml     # 3 new operations, 3 changed, new schemas, examples
├── checklists/requirements.md
└── tasks.md             # Phase 2 (/speckit-tasks; not created here)
```

### Source Code

New files are marked `+`, changed `~`, removed `-`.

**`hl-budget-backend`** (Phase A)

```text
src/main/kotlin/com/valyalkin/hlbudget/
├── controller/
│   ├── TotalsController.kt          + GET /monthly-totals, /category-totals
│   ├── MerchantController.kt        + GET /merchants
│   └── SpendingController.kt        ~ docs for id-or-name; examples
├── dto/
│   ├── MoneyTotalDto.kt             +
│   ├── totals/   MonthlyTotalsResponse, CategoryTotalsResponse, TotalsMappings  +
│   ├── merchant/ MerchantListResponse, MerchantMappings                         +
│   └── spending/SpendingRequest.kt  ~ optional ids + names; exactly-one validation
├── model/
│   ├── MonthTotal.kt, CategoryTotal.kt, Merchant.kt                             +
│   └── SpendingCommand.kt           ~ CategoryRef/AccountRef = ById | ByName
├── repository/
│   ├── spending/TotalsQueries.kt    + monthly and per-category GROUP BY (native, numeric)
│   ├── spending/MerchantQueries.kt  + DISTINCT ON latest per lower(trim(merchant)), counts, limit
│   └── account/AccountRepository.kt ~ lock all of a user's accounts; count
├── service/
│   ├── TotalsService.kt             + fills empty months; today from Clock + zone
│   ├── MerchantService.kt           +
│   ├── NamedItemResolver.kt         + find-or-create category/account by name (shared rules)
│   ├── SpendingService.kt           ~ resolve refs before save; fingerprint on resolved ids
│   ├── AccountService.kt            ~ last-account refusal
│   └── NamedItemErrors.kt           ~ lastAccount()
└── config/OpenApiConfig.kt          ~ tags Totals, Merchants
src/test/kotlin/com/valyalkin/hlbudget/
├── service/   TotalsServiceTest, MerchantServiceTest, NamedItemResolverTest,
│              SpendingServiceTest ~, AccountServiceTest ~                        + / ~
└── controller/ MonthlyTotalsIT, CategoryTotalsIT, MerchantsIT, SpendingByNameIT,
               LastAccountIT, TotalsPerformanceIT, OwnershipIT ~                  + / ~
contracts/api.yaml                   ~ regenerated
README.md                            ~ new endpoints
```

**`hl-budget-frontend`** (Phase B)

```text
next.config.ts                       ~ redirects(): /home → /spendings (permanent)
proxy.ts                             ~ PROTECTED_PREFIXES = ['/spendings']
contracts/api.yaml, app/lib/api-schema.d.ts      ~ regenerated
app/
├── page.tsx                         ~ signed-out screen; redirects to /spendings with a session
├── layout.tsx                       ~ new fonts and ground colour
├── globals.css                      ~ @theme tokens from the design (R-012)
├── home/                            - removed (page, loading, error, tests)
├── spendings/
│   ├── page.tsx                     + sync shell; <Suspense> around the month section
│   ├── loading.tsx                  + skeleton
│   └── error.tsx                    + error boundary with retry (FR-092)
├── lib/
│   ├── api.ts                       ~ problem `details`/`errors` into ApiError (R-007)
│   ├── definitions.ts               ~ aliases: Spending, Account, Category, MonthlyTotals,
│   │                                  CategoryTotals, Merchant; UI types (Draft, Suggestion)
│   ├── data.ts                      ~ getMonthlyTotals, getCategoryTotals, getMonthSpendings
│   │                                  (all pages), getCategories, getAccounts, getMerchants
│   ├── actions.ts                   + saveSpending, deleteSpending, create/delete Category/Account
│   ├── messages.ts                  + API problem → spec wording
│   └── utils.ts                     ~ cents, money format, share, month params and bounds,
│                                      rankSuggestions, donut arcs, draft validation
└── ui/
    ├── fonts.ts                     ~ Bricolage Grotesque, IBM Plex Sans, IBM Plex Mono
    ├── skeletons.tsx                ~ SpendingsSkeleton
    ├── app-header.tsx               + app name, Sign in | Sign out + user icon (opens Settings)
    ├── home/, landing/              - replaced (tests moved or removed)
    ├── spendings/
    │   ├── month-section.tsx        + async: awaits searchParams, loads data in parallel
    │   ├── month-line.tsx           + year arrows, 12 month links with totals, future disabled
    │   ├── month-total-card.tsx     +
    │   ├── spendings-table.tsx      + 'use client': open row, hover/focus actions, delete
    │   ├── spending-row.tsx         + read-only row with edit/remove buttons
    │   ├── entry-row.tsx            + 'use client': form, Tab/Enter rules, merchant fill-in
    │   ├── combobox.tsx             + 'use client': ARIA combobox with inline autofill
    │   ├── date-picker.tsx          + 'use client': popover calendar, Today, future disabled
    │   ├── by-category-card.tsx     + 'use client': Table/Pie toggle (persisted view)
    │   ├── category-table.tsx       +
    │   └── category-pie.tsx         + SVG donut + legend, no cap
    └── settings/
        ├── settings-dialog.tsx      + 'use client': <dialog>, vertical tabs
        ├── categories-panel.tsx     +
        └── accounts-panel.tsx       +
   (each component with a colocated *.test.tsx)
e2e/
├── support.ts                       ~ seed/cleanup helpers through the UI
├── month-view.spec.ts               + US1
├── add-spending.spec.ts             + US2
├── keyboard-autofill.spec.ts        + US3
├── edit-delete.spec.ts              + US4
├── by-category.spec.ts              + US5
├── settings.spec.ts                 + US6
├── sign-in.spec.ts, sign-out.spec.ts, protected-routes.spec.ts   ~ US7, new routes
└── a11y.spec.ts                     + axe scans (SC-008)
mocks/handlers.ts                    ~ MSW handlers for the new operations
```

**Structure Decision**: This keeps the existing one-service-per-repo layout. The backend follows
the `backend-developer` package roles, with feature subpackages only in `dto/`. The frontend
follows the `frontend-developer` layout:
- a route segment `app/spendings/`;
- feature component folders `app/ui/spendings/` and `app/ui/settings/`;
- all reads in `data.ts` and all writes in `actions.ts`.

There are no new top-level directories.

## Delivery order

1. **Branches**: create `003-spending-screens` from `origin/main` in both worktrees (R-014).
2. **Phase A, backend** (blocks the frontend's types):
   - A1: `NamedItemResolver`, plus spending create and update by name (US2).
   - A2: last-account rule (US6).
   - A3: monthly and category totals (US1, US5).
   - A4: merchants (US3).
   - A5: contract export, `./gradlew check`, PR, release.
3. **Phase B, frontend foundation**:
   - regenerate types;
   - `ApiError` details, `messages.ts`, `utils.ts`;
   - tokens and fonts, the app header;
   - routes, proxy and redirect;
   - `/` as the signed-out screen.
4. **US1** month view: month section, month line, table read-only, footer, total card, empty and
   loading states.
5. **US2** add: entry row, date picker, `saveSpending`, navigate to the saved month.
6. **US3** keyboard and autofill: combobox, ranking, merchant fill-in, Tab/Enter rules.
7. **US4** edit and delete: row actions on hover and focus, edit mode, `deleteSpending`.
8. **US5** By category: toggle, table, donut.
9. **US6** Settings: dialog, panels, actions, messages.
10. **US7** header sign-in/out polish, the e2e updates for the new routes, the axe spec.
11. **Gates and deploy**: pnpm gates and `pnpm e2e`, then frontend PR, release and
    `hl-deploy-status` (quickstart §7).

US1 can start as soon as the A3 types exist. US2 needs A1. US3 needs A4 and US2. US4 needs US2.
US5 needs A3. US6 needs A2.

## Complexity Tracking

| Violation | Why Needed | Simpler Alternative Rejected Because |
|-----------|------------|-------------------------------------|
| `GET /monthly-totals`, `/category-totals` and `/merchants` are not paginated (Principle VII: "Collection endpoints MUST be paginated") | Monthly totals always return 12 rows. Category totals return at most the user's category count. Merchants are capped by `limit` (default 500, at most 1000) and report `truncated`. Each screen needs the whole set at once | Paging a fixed 12-row result, or the autofill source list, would make every client loop for no gain. Each response uses a named `items`/`months` field, so paging parameters can be added later as optional |
| The inline entry row needs JavaScript for autofill, the calendar, Tab/Enter rules and hover actions (the `frontend-developer` skill: "forms must work without JavaScript") | FR-050–FR-058 and FR-031 describe keyboard and pointer behaviour that only script can provide | The row is still a real `<form action>` with named inputs calling a Server Action, so a basic submit works without script. Building a separate no-JS add page duplicates the flow for a case this personal app does not have |
