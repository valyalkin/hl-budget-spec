# Implementation Plan: Mobile View

**Branch**: `006-mobile-view` | **Date**: 2026-10-31 | **Spec**: [spec.md](./spec.md)

**Input**: Feature specification from `/specs/006-mobile-view/spec.md`

## Summary

HL Budget gets a phone layout for screens under 640px wide. It is a touch-first presentation of
the same Spendings screen: Month and Year dropdowns, a compact list with a floating "Add
spending" button, a By category Table / Pie, full-screen Edit / Add / Fix row, Settings and
Import screens, and bottom sheets for the account menu and the delete confirmation. **The
desktop layout must not change** (FR-002), and that drives the approach.

**Frontend only** (`hl-budget-frontend`):
- **Layout switch (R-001, R-002)**: The server keeps rendering the desktop tree exactly as
  today, inside a `DesktopOnly` wrapper (`display: contents` at 640px and wider). A
  `PhoneOnly` sibling shows a phone skeleton below 640px. After hydration, a `matchMedia` hook
  unmounts whichever tree does not apply and swaps the trees if the width crosses 640px.
- **New phone components** under `app/ui/phone/`, written from scratch. No desktop component
  is imported or edited. They reuse the pure helpers in `app/lib/`:
  - validation (`draftProblems`) and suggestion ranking (`rankSuggestions`);
  - CSV reading (`checkFile`, `readImport`);
  - money, date, calendar and donut helpers.
- **Existing Server Actions, unchanged**: `saveSpending`, `deleteSpending`, the category and
  account actions, and `importSpendings`.
- **Native `<dialog>` for every screen and sheet (R-005)**, with history entries so the phone
  back gesture closes the top layer (R-006, clarified).
- **One new read helper**, `getEarliestSpendingYear()`, which calls an existing endpoint
  (R-004), and one new pure helper, `topCategories()`, for the top-6 pie (R-013).
- **A second Playwright project, `phone`**, for `e2e/phone/`. The existing desktop project
  ignores that folder, so desktop specs run exactly as before (R-016).

**Backend**: no change. There are no new endpoints, migrations, environment variables,
dependencies or services.

## Technical Context

**Language/Version**: TypeScript 5.9 (strict), Node 22, React 19, Next.js 16 App Router
(`cacheComponents`, React Compiler). All unchanged.

**Primary Dependencies**: Existing only: Next.js, React, Tailwind CSS 4, `@auth0/nextjs-auth0`,
zod, clsx. No new runtime or dev dependency.

**Storage**: N/A in the frontend. Server data is unchanged (Postgres behind the backend).

**Testing**:
- Vitest + Testing Library + MSW for unit tests.
- Playwright for e2e: the existing `chromium` desktop project, plus a new `phone` project
  (`iPhone 13` descriptor on Chromium), with `@axe-core/playwright` scans.

**Target Platform**:
- Phone browsers below 640px: iOS Safari 17+ and Android Chrome (current).
- Desktop browsers: unchanged.

**Project Type**: Web application. The change is to the frontend service only.

**Performance Goals**:
- A spending can be recorded in under 20 seconds on a phone (SC-001).
- No extra round trip on a month change compared with the desktop.
- The phone skeleton is replaced as soon as hydration completes.

**Constraints**:
- Zero change to desktop behaviour, wording or layout at 640px and wider (FR-002, SC-005).
- Touch targets of at least 44px, input text of at least 16px, no hover-only actions
  (FR-004 – FR-006).
- No sideways scroll from 320 to 639px (FR-007).
- Content Security Policy unchanged (no inline scripts added; `'unsafe-inline'` styles are
  already allowed).

**Scale/Scope**: One user per account, months of up to a few hundred spendings, CSV imports of
up to 1,000 rows. About 6 phone screens and 2 sheets; roughly 15 new components and 2 new
helpers.

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

| Principle | Check | Status |
|---|---|---|
| I. Spec-driven, foundation-first | Spec and clarifications are done; this is an independently deployable, user-visible increment on the working app | ✅ Pass |
| II. Separate services, contract-first | Frontend renders UI only. It adds no business rules: validation and CSV helpers are the existing ones, and the server stays authoritative. It reads through existing server-side loaders and Server Actions. No hand-written API types, and the OpenAPI contract is unchanged | ✅ Pass |
| III. Secure by default | Same route, same Auth0 session and token custody; nothing new reaches the browser except data the desktop already sends. `/spendings` stays `noindex`. The CSV file is read in the browser and never stored, as in 004 | ✅ Pass |
| IV. Tests and quality gates | Unit tests for every new component and helper. A Playwright spec for every new phone flow (US1–US6, back gesture). Axe scans on every phone screen. Gate: `pnpm typecheck && pnpm lint && pnpm test`, plus both e2e projects. Existing tests are not edited | ✅ Pass |
| V. Homelab-ready operations | Container, ports, probes, configuration and deploy flow are unchanged; the release follows the usual workflow → hl-helm PR → Argo CD | ✅ Pass |
| VI. Simplicity at personal scale | No new services, dependencies or caches. The one deliberate duplication (phone versions of the calendar and suggestions instead of reusing desktop components) is listed under Complexity Tracking | ✅ Pass (justified) |
| VII. Versioned, consistent API | No API change. One new caller of an existing v1 endpoint with an existing sort option | ✅ Pass |

**Post-design re-check (after Phase 1)**: unchanged; all gates pass. The design adds no
endpoint, stored data or dependency. The desktop guard (R-001, R-016) satisfies FR-002 without
touching desktop components or their tests.

## Project Structure

### Documentation (this feature)

```text
specs/006-mobile-view/
├── plan.md              # This file
├── research.md          # Phase 0: R-001 – R-016
├── data-model.md        # Phase 1: client-side state, screen transitions
├── quickstart.md        # Phase 1: run and check (phone, desktop regression)
├── contracts/
│   └── phone-ui.md      # Phase 1: UI contract (roles, names, texts); no API change
├── checklists/
│   └── requirements.md  # Spec quality checklist
└── tasks.md             # Phase 2 (/speckit-tasks; not created here)
```

### Source Code (hl-budget-frontend)

```text
app/
├── layout.tsx                      # + viewport export: viewportFit 'cover', interactiveWidget (R-014)
├── globals.css                     # + phone-only rules: scroll lock for dialog[data-phone], safe-area vars
├── page.tsx                        # unchanged (signed-out route)
├── spendings/
│   ├── page.tsx                    # wraps existing header/notice/section in <DesktopOnly>; adds
│   │                               #   <PhoneUiProvider> + <PhoneOnly><Suspense><PhoneSection/></Suspense>
│   └── loading.tsx                 # unchanged
├── lib/
│   ├── data.ts                     # loadSpendingsPage wrapped in React cache; + getEarliestSpendingYear (R-004)
│   ├── phone.ts                    # NEW pure helpers: topCategories, yearOptions, monthOptions,
│   │                               #   PHONE_FIELDS order, firstPhoneProblem, importRowDisplay
│   └── phone.test.ts               # NEW
└── ui/
    ├── signed-out.tsx              # max-sm: variants only, if needed (R-015)
    └── phone/                      # NEW, all phone components (+ *.test.tsx each)
        ├── use-is-phone.ts         # matchMedia('(width < 40rem)') via useSyncExternalStore
        ├── layout-switch.tsx       # DesktopOnly / PhoneOnly wrappers
        ├── phone-skeleton.tsx      # pre-hydration placeholder (sm:hidden)
        ├── phone-ui-provider.tsx   # screen/sheet state machine, notice, view choices, history depth (R-005, R-006)
        ├── phone-section.tsx       # server: cached loadSpendingsPage + earliest year → PhoneApp
        ├── phone-app.tsx           # client root (data-layout="phone"): header, picker, list, breakdown, screens
        ├── phone-header.tsx
        ├── month-year-picker.tsx   # two native <select>s → router.push(?month=) (R-003)
        ├── spending-list.tsx       # rows + empty state + floating Add button
        ├── category-breakdown.tsx  # Table | Pie, top-6 donut + legend (R-013)
        ├── phone-dialog.tsx        # shared full-screen / bottom-sheet <dialog> with history entry
        ├── spending-form.tsx       # Edit / Add / Fix row (modes, validation order, bottom bar)
        ├── phone-calendar.tsx      # inline calendar (R-009)
        ├── suggest-input.tsx       # type-or-pick with ranked 44px options (R-007)
        ├── delete-sheet.tsx        # confirmation bottom sheet
        ├── account-menu.tsx        # Settings / Import CSV / Sign out sheet
        ├── phone-settings.tsx      # Categories | Accounts tabs
        ├── phone-import.tsx        # choose-file step
        ├── phone-review.tsx        # review list, banner, Add N spendings
        └── use-keyboard-inset.ts   # visualViewport → --keyboard-inset (R-014)

e2e/
├── phone/                          # NEW: one spec per story + shared checks
│   ├── support.ts                  # signIn at 390px, assertTouchTargets, assertNoZoomInputs, assertNoSideScroll
│   ├── month-view.spec.ts
│   ├── add-spending.spec.ts
│   ├── edit-delete.spec.ts
│   ├── by-category.spec.ts
│   ├── menu-settings.spec.ts
│   ├── import-csv.spec.ts
│   ├── back-gesture.spec.ts
│   └── a11y.spec.ts
└── *.spec.ts                       # existing desktop specs: NOT edited

playwright.config.ts                # + 'phone' project (testMatch e2e/phone/); 'chromium' gets testIgnore
README.md                           # + phone layout and the phone e2e project
```

**Structure Decision**:
- Frontend only, inside the existing App Router layout.
- **New code**: all phone UI goes in `app/ui/phone/` and phone helpers in `app/lib/phone.ts`.
- **Existing files that change**: these are limited to wiring, so desktop rendering stays
  as it is:
  - `spendings/page.tsx` (wrappers);
  - `layout.tsx` (viewport meta);
  - `globals.css` (phone-scoped rules);
  - `data.ts` (React `cache` wrapper plus one new read);
  - `signed-out.tsx` (`max-sm:` only);
  - `playwright.config.ts` (projects).

## Complexity Tracking

| Choice | Why Needed | Simpler Alternative Rejected Because |
|---|---|---|
| Phone versions of the calendar and suggestion input instead of reusing the desktop `DatePicker` and `Combobox` | FR-002 forbids changing desktop behaviour; the desktop pieces are keyboard-first popovers (inline completion, grid arrow keys) and the phone needs inline, tap-first versions | Making the desktop components configurable would edit desktop code and risk regressions in its keyboard rules; the shared logic already lives in `app/lib` and is reused |
| Two sibling trees with a client-side switch (`DesktopOnly` / `PhoneOnly`) | The server cannot know the viewport, and the desktop header must stay outside `<Suspense>` | UA sniffing changes desktop streaming and can't tell portrait from landscape; rendering both full trees duplicates content and risks the desktop e2e suite |
| One extra small read per page load (`getEarliestSpendingYear`) | The Year dropdown range (FR-013) needs the earliest year, and the server can't tell which layout will show | Fetching it from the client after mount makes the Year options flicker |
