# Implementation Plan: Overview tab

**Branch**: `011-overview-tab` | **Date**: 2026-11-08 | **Spec**: [spec.md](./spec.md)

**Input**: Feature specification from `/specs/011-overview-tab/spec.md`

## Summary

An Overview tab after Income, on desktop and phone, showing one calendar year: Earned, Spent and
Saved tiles; an Earned vs spent card (column chart or table by month); and Spent by category and
Spent by account cards (ranked bar chart or table). The charts use ECharts.

**Frontend only.** The backend is not changed (research R-001): the year is built from the
existing monthly spending and income totals and the existing category and account totals, asked
for January 1 to December 31.

**Frontend** (`hl-budget-frontend`):
- New route `/overview?month=&year=`. `month` is passed through for the other tabs and `year` is
  the shown year, with arrows as replace-links (R-002).
- Tabs become Spendings | Income | Overview, through a `Tab` type and `TABS` list, used by the
  desktop nav and the phone segmented switch (R-003).
- `loadOverview` in `app/lib/overview.ts` makes two rounds of parallel API calls and builds
  `OverviewData`: tiles, 12 month rows, breakdown rows with shares (data-model.md).
- `echarts@^6` from `echarts/core` (bar chart, grid, tooltip, SVG renderer), loaded only on
  Overview through one `EChart` client wrapper. Pure option builders with keyboard, tap and
  narrow-width behaviour (R-004 – R-006).
- One responsive content tree. The desktop gets `AppHeader`; the phone gets `PhoneHeader` and an
  account menu without Import CSV (R-008). Chart / Table choices are kept in `localStorage`
  (R-007).

## Technical Context

**Language/Version**: TypeScript 5 strict (frontend). Backend Kotlin 2 / Java 21, unchanged.

**Primary Dependencies**: Next.js 16 App Router, React 19, Tailwind v4, zod. New: **echarts 6**
(user's choice; tree-shaken core import).

**Storage**: none new. `localStorage` holds three view-choice keys in the browser.

**Testing**: Vitest + Testing Library for the data builder, formatting, option builders and
components (with the `EChart` wrapper mocked); Playwright desktop and phone projects with axe for
the flows.

**Target Platform**: homelab Kubernetes (Argo CD); browsers on desktop and phone.

**Project Type**: web application. Two services in two repositories, with this spec repo
alongside. Only the frontend changes.

**Performance Goals**: Overview renders a year with up to 3,000 entries in under 2 s and changes
year in under 1 s (SC-003). That is at most two rounds of parallel calls, each a single SQL
aggregate on an indexed `(user_id, date)` range. ECharts loads only on `/overview`.

**Constraints**: Spendings and Income behave exactly as before, except for the third tab.
`LEDGERS` stays two items; `AccountMenu` and `PhoneHeader` defaults are unchanged. No CSP change
(R-004). Phone: no sideways page scroll from 360 px, 44 px touch targets.

**Scale/Scope**: one user (a handful at most); 12 months, about 20 categories and a few accounts
per year.

All unknowns are resolved in [research.md](./research.md).

## Skills that drive the implementation

| Skill | Owns | Used in |
|---|---|---|
| `frontend-developer` | Route, data loader, tabs, ECharts wrapper and option builders, cards, phone shell, unit tests, e2e, pnpm gates | Phases 1–6 |
| `homelab:hl-deploy-status` | Verify the frontend rollout | Deploy |

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-checked after Phase 1 design.*

| Principle | Check | Result |
|---|---|---|
| I. Spec-driven | Spec 011 → plan → tasks before code | Pass |
| II. Separate services, contract-first | Sums stay in the backend's existing totals endpoints. The frontend adds 12 monthly decimals in cents and computes display shares and averages, as the month total card and breakdown tables already do. No hand-written API types: the existing generated types are used and the contract is unchanged | Pass |
| III. Secure by default | `/overview` uses the existing session guard and calls the API from the server only; `robots: noindex`; no new data exposure, every endpoint is already scoped to the caller | Pass |
| IV. Tests and gates | Unit tests for `loadOverview` (clamping, sums, nulls for future months, shares), formatting, option builders and aria summaries, tabs and cards; Playwright for US1–US5 on desktop and phone with axe; existing specs green. `pnpm typecheck && pnpm lint && pnpm test` and `pnpm e2e` must pass | Pass |
| V. Homelab-ready | No new config, secret or migration; deploy through release → hl-helm → Argo CD | Pass |
| VI. Simplicity | No new service or endpoint (R-001). One new dependency, ECharts, requested by the user and justified below; one responsive tree instead of phone/desktop twins (R-008) | Pass with justification |
| VII. Versioned API | No API change | Pass (not affected) |

**Post-design re-check**: unchanged. No contract delta, so backward compatibility is
**compatible** with nothing to check on the API.

## Project Structure

### Documentation (this feature)

```text
specs/011-overview-tab/
├── plan.md
├── research.md
├── data-model.md
├── quickstart.md
├── contracts/overview-ui.md
├── checklists/requirements.md
└── tasks.md              # /speckit-tasks
```

### Source Code

```text
hl-budget-frontend/
├── package.json                                        # + echarts ^6
├── app/overview/page.tsx, loading.tsx, error.tsx       # new route (R-002, R-010)
├── app/lib/
│   ├── ledger.ts                                       # + Tab, TABS (label, route) (R-003)
│   ├── overview.ts                                     # new: parseYear, loadOverview → OverviewData (data-model.md)
│   ├── overview-charts.ts                              # new: monthsOption, breakdownOption, aria summaries,
│   │                                                   #      narrow-width label rules (R-005, R-006)
│   └── utils.ts                                        # + formatWholeDollars, formatShare (R-009)
├── app/ui/
│   ├── ledger-tabs.tsx                                 # iterate TABS; current: Tab; Overview link keeps ?month=
│   ├── app-header.tsx                                  # StaticTabs over TABS; ledger prop → current: Tab
│   ├── load-error.tsx                                  # accept the Overview message (tab instead of ledger)
│   ├── charts/echart.tsx                               # new: client wrapper: dynamic import, init, resize,
│   │                                                   #      dispose, keyboard showTip, tap-outside hideTip
│   ├── overview/                                       # new
│   │   ├── overview-page.tsx                           # shell: DesktopOnly AppHeader / PhoneOnly PhoneHeader + menu
│   │   ├── overview-section.tsx                        # server: loadOverview → <OverviewView>
│   │   ├── overview-view.tsx                           # client: heading, period, year picker, tiles, cards
│   │   ├── year-picker.tsx                             # previous / next links
│   │   ├── total-tiles.tsx                             # Earned, Spent, Saved
│   │   ├── chart-card.tsx                              # card frame with Chart / Table switch + stored choice
│   │   ├── months-card.tsx, breakdown-year-card.tsx    # Earned vs spent; Spent by category / account
│   │   ├── use-view-choice.ts                          # localStorage hook (R-007)
│   │   └── overview-skeleton.tsx
│   └── phone/
│       ├── phone-header.tsx                            # optional current: Tab prop
│       └── account-menu.tsx                            # allowImport prop (default true)
└── e2e/overview/*.spec.ts, e2e/phone/overview/*.spec.ts  # new; a11y specs cover /overview
```

**Structure Decision**: same repository layout. Overview files sit under `app/ui/overview/` and
`app/lib/overview*.ts`. Shared components get small optional props whose defaults keep
Spendings and Income unchanged.

## Delivery order

1. **Branch**: `011-overview-tab` from `origin/main` in `hl-budget-frontend` (this spec branch
   exists). No backend branch.
2. **Phase 1, foundation**: add `echarts`; `Tab`/`TABS`; tabs on desktop and phone show Overview
   (existing tab e2e updated for three tabs); `formatWholeDollars`, `formatShare`;
   `parseYear` + `loadOverview` with unit tests; the `/overview` route with skeleton, error page
   and an empty shell.
3. **US1 tiles (P1)**: heading, period line, Earned / Spent / Saved with notes; e2e for sums,
   negative Saved, empty year.
4. **US2 Earned vs spent (P1)**: `EChart` wrapper, `monthsOption`, card with Chart / Table and
   stored choice, table with Net; keyboard and tooltip; e2e.
5. **US3 breakdowns (P2)**: `breakdownOption`, two cards, dynamic height, empty states; e2e
   including "income never counted".
6. **US4 year picker (P2)**: links with clamping and disabled ends; `?month=` round trip to the
   other tabs; e2e.
7. **US5 phone (P2)**: phone shell (header with three options, menu without Import CSV),
   narrow-width chart rules, tap-outside hide, table scroll; phone e2e at 390 and 360 wide;
   axe on all views.
8. **Gates and deploy**: pnpm gates, `pnpm e2e`, `pnpm build` (check chunks), PR, release,
   `hl-deploy-status`, live check (quickstart §5), README Features update via a small PR.

## Complexity Tracking

| Violation | Why Needed | Simpler Alternative Rejected Because |
|---|---|---|
| New frontend dependency `echarts` (Principle VI: added dependencies must be justified) | The user asked for ECharts for all Overview charts; grouped columns, ranked bars, axes, tooltips with tap and resizing would otherwise be hand-built in SVG | Hand-built SVG (like the donut) would need its own scales, axes, tooltips and resize handling for two chart types; the tree-shaken core import loaded only on `/overview` keeps the cost off the other tabs |
