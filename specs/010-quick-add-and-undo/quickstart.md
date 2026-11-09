# Quickstart: Verify quick add and undo

## Prerequisites

- `hl-budget-backend` running locally (or the dev stack used by e2e), signed in with a test user
  who has at least one account and category.
- `hl-budget-frontend` on branch `010-quick-add-and-undo`, `pnpm install` done.

## Gates

```bash
cd hl-budget-frontend
pnpm typecheck && pnpm lint && pnpm test
pnpm exec playwright test e2e/quick-add-undo.spec.ts e2e/edit-delete.spec.ts e2e/income/add-edit-delete.spec.ts
pnpm exec playwright test e2e/phone   # phone behaviour unchanged (SC-005)
```

All must pass.

## Manual checks (desktop width)

Run each check on **Spendings**, then repeat on **Income**.

1. **Quick add by mouse**: Hover a row's date and check for a pointer cursor and the tooltip
   "Add on Oct 14". Click it. Expect the new row at the top with date Oct 14, the calendar
   closed and the cursor in merchant. Fill in the rest, press Enter, and check the entry is
   listed under Oct 14 with the view still on the same month.
2. **Quick add by keyboard**: Tab to a row date, press Enter, and expect the same result.
3. **Locked while open**: With an add or edit row open, other dates are plain text and clicking
   them does nothing.
4. **Undo**: Click remove. The row fades, Undo shows without hovering, and focus is on Undo.
   Press Enter within 5 s. The row is back, and after a reload it is still there.
5. **Delete after wait**: Click remove and wait. The row disappears at about 5 s and the footer
   total and cards drop. After a reload it is still gone.
6. **Several at once**: Remove three rows about a second apart. Undo the middle one. The other
   two disappear about 5 s after their own clicks.
7. **Leaving flushes**: Remove a row and switch month within 5 s. Come back and check it is gone.
   Repeat, opening Settings instead of switching month.
8. **Unload keeps**: Remove a row and reload within 5 s. The row is still there. Repeat with
   Sign out, then sign back in.
9. **Failure**: Stop the backend, remove a row and wait 5 s. The row returns to normal with the
   "not deleted" message.
10. **Phone**: At 390 px width, deleting still asks for confirmation in the bottom sheet, and
    there is no quick add.
