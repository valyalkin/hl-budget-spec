# Implementation Plan: Quick add from a date, and undo for delete

**Branch**: `010-quick-add-and-undo` | **Date**: 2026-11-08 | **Spec**: [spec.md](spec.md)

**Input**: Feature specification from `specs/010-quick-add-and-undo/spec.md`

## Summary

Two changes to the shared desktop entry table (`SpendingsTable`, used by both the Spendings and
Income tabs through the ledger context):

1. A saved row's date becomes a button ("Add on Oct 14") that opens the add row with that date,
   the calendar closed and focus in the merchant field.
2. Remove starts a 5-second pending delete held in the browser. The row fades and shows Undo.
   When the timer ends the existing `deleteSpending` server action runs once. Leaving the
   screen within the app (the table unmounts, or Settings / Import opens) runs all pending
   deletes at once; a full page unload simply drops them, so the entries are kept.

Frontend only. No backend, API contract or database change.

## Technical Context

**Language/Version**: TypeScript 5.9 (`strict`), React 19, Next.js 16 App Router

**Primary Dependencies**: Existing only: Next.js server actions (`deleteSpending`), Tailwind. No
new packages.

**Storage**: N/A. Pending deletes live in React state on the open page only.

**Testing**: Vitest + Testing Library with fake timers for unit tests; Playwright for e2e on
both the Spendings and Income tabs (`e2e/edit-delete.spec.ts`, `e2e/income/add-edit-delete.spec.ts`
updated, and a new `e2e/quick-add-undo.spec.ts`)

**Target Platform**: Desktop browsers (desktop layout only; the phone layout is untouched)

**Project Type**: Web application. This feature touches only the frontend repository
(`hl-budget-frontend`).

**Performance Goals**: The row fades and Undo shows within one frame of the click. The delete is
sent 5 s after remove (±100 ms), and totals update within 1 s of it finishing (spec SC-003).

**Constraints**: No hover-only Undo. Each delete is sent at most once. The phone view and the
backend do not change. The `jsx-a11y` lint rules must pass (the date becomes a real `<button>`).

**Scale/Scope**: One table component plus one row component, one new hook, and small changes to
`EntryRow`, the Settings dialog and the Import dialog. About 6 source files and 4 test files.

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

| Principle | Status | Notes |
|-----------|--------|-------|
| I. Spec-driven | PASS | Spec 010 written and updated before the plan. |
| II. Separate services, contract-first | PASS | No contract change. The frontend still calls the backend only through the existing server action, and business rules stay in the backend. The 5 s wait is UI timing, not a business rule. |
| III. Secure by default | PASS | No new data paths. Tokens stay server-side because the delete still goes through the server action. |
| IV. Tests and quality gates | PASS (planned) | Unit tests for the hook, row and table with fake timers. Playwright specs for every new flow on both tabs. Gate: `pnpm typecheck && pnpm lint && pnpm test`, plus e2e. |
| V. Homelab-ready operations | PASS | No container, config or deploy change beyond the normal frontend release. |
| VI. Simplicity | PASS | No new dependency, no server-side soft delete, and no toast system. One small hook. |
| VII. API design | N/A | No endpoint added or changed. The contract is unchanged and backward compatible. |

**Contract change statement**: none. `DELETE /api/v1/spendings/{id}` and
`DELETE /api/v1/income/{id}` are used exactly as today.

Post-design re-check (after Phase 1): still PASS. The design adds no services, dependencies or
endpoints.

## Project Structure

### Documentation (this feature)

```text
specs/010-quick-add-and-undo/
├── plan.md              # This file
├── research.md          # Phase 0: decisions
├── data-model.md        # Phase 1: pending-delete state machine, OpenRow change
├── quickstart.md        # Phase 1: how to verify
├── contracts/
│   └── ui-entry-table.md   # Phase 1: UI contract (roles, names, timings, events)
├── checklists/requirements.md
└── tasks.md             # Phase 2 (/speckit-tasks)
```

### Source Code (`hl-budget-frontend`)

```text
app/
├── lib/
│   └── pending-deletes.ts          # NEW: FLUSH_DELETES_EVENT name + flushPendingDeletes()
├── ui/
│   ├── spendings/
│   │   ├── use-pending-deletes.ts  # NEW: hook: start / undo / flush, 5 s timers, send-once
│   │   ├── use-pending-deletes.test.ts   # NEW
│   │   ├── spendings-table.tsx     # CHANGED: OpenRow add carries a date; uses the hook;
│   │   │                           #   status live region; replaces the single `removing` state
│   │   ├── spendings-table.test.tsx      # CHANGED
│   │   ├── spending-row.tsx        # CHANGED: date button; pending look with Undo
│   │   ├── spending-row.test.tsx   # NEW
│   │   ├── entry-row.tsx           # CHANGED: `openCalendar` / `startField` props
│   │   └── entry-row.test.tsx      # CHANGED
│   ├── settings/settings-dialog.tsx  # CHANGED: flushPendingDeletes() when it opens
│   └── import/import-dialog.tsx      # CHANGED: flushPendingDeletes() when it opens
e2e/
├── quick-add-undo.spec.ts          # NEW: both ledgers
├── edit-delete.spec.ts             # CHANGED: delete now waits 5 s / uses Undo
└── income/add-edit-delete.spec.ts  # CHANGED: same
```

**Structure Decision**: Frontend-only change in the existing `app/ui/spendings` folder, which
already serves both ledgers through `useLedger()`. No backend work. The phone components
(`app/ui/phone/*`) are not touched.

## Design Overview

- **Quick add**: `OpenRow` becomes `{ mode: 'add'; date?: string } | { mode: 'edit'; … }`.
  `SpendingRow` gets `onAddOnDate()`. Its date cell renders a `<button>` when the row is neither
  locked nor pending, and plain text otherwise. The add `EntryRow` gets
  `initial.date = open.date ?? defaultDateFor(ym, today)`, `openCalendar={!open.date}` and
  `startField={open.date ? 'merchant' : 'date'}`.
- **Pending deletes**: `usePendingDeletes({ onDelete })` keeps
  `Map<id, 'waiting' | 'sending'>` in state and the timers in a ref. Its actions:
  - `start(id)` creates a 5 s timer.
  - `undo(id)` works only while `waiting`. It clears the timer and drops the entry.
  - When the timer fires, or on `flushAll()`, the entry moves to `sending` and `onDelete(id)` is
    called once (a ref set guards against double sends). On `ok`, or on `gone` (404), the entry
    stays `sending` until the refreshed list no longer contains the row, which avoids a flash
    back to full opacity. On `error` it is dropped and the table shows the message. On
    `signed-out` the page goes to `/`.
  - On unmount, which happens on a month or year change (the table is keyed by month) or a tab
    change (a route change), it calls `flushAll()`. The server action keeps running after
    unmount, and its revalidation refreshes the next screen.
  - It listens for `FLUSH_DELETES_EVENT`, which the Settings and Import dialogs dispatch when
    they open.
  - A full page unload (reload, close, Sign out link) kills the timers, so nothing is sent.
- **Announcements**: a separate `sr-only` `aria-live="polite"` status region in the table, apart
  from the error line, announces "{merchant} on {date} removed. Undo available for 5 seconds.",
  "{merchant} on {date} restored." and "{merchant} on {date} deleted."
- **Focus**: remove moves focus to the row's Undo button (after render). Undo returns focus to
  the row's remove button.

## Complexity Tracking

No violations. Nothing to justify.
