# UI Contract: Desktop entry table (Spendings and Income)

This feature exposes no new API. This contract fixes the user-facing interface that the unit
and Playwright tests rely on. `{date}` is the table's short date (for example `Oct 14`).
`{party}` is the merchant on Spendings and the source on Income (the ledger's party text).

## Quick add

| Element | Role | Accessible name | Visible when | Activation |
|---------|------|-----------------|--------------|------------|
| Row date | `button` | `Add on {date}` | no row open and the row is not pending | click, Enter, Space |

After activation:

- One new entry row is at the top of the table body.
- Its date button shows `{date}`.
- The calendar (`dialog` / popup) is not open.
- `document.activeElement` is the merchant/party combobox input.

While any row is open, or the row is pending, the date is plain text: there is no `button`
named `Add on {date}`.

## Remove and Undo

| Element | Role | Accessible name | Visible when |
|---------|------|-----------------|--------------|
| Remove | `button` | `Remove {party} on {date}` (unchanged) | row normal (hover/focus as today) |
| Undo | `button` | `Undo` (title `Undo remove`) | row pending, always visible (no hover needed) |

Timings, measured from the remove click:

- Within one frame: the row has reduced opacity (the row element carries `data-pending="true"`
  for tests), Remove and Edit are replaced by Undo, and focus is on Undo.
- From 0 to 5 000 ms: Undo restores the row (`data-pending` removed), no request is sent, and
  focus goes to Remove.
- At 5 000 ms: exactly one `DELETE` request for the entry is sent, through the existing server
  action, and Undo becomes disabled.
- On success, or if the entry is already gone: the row leaves the table, the footer, month line,
  total card and breakdown cards update, and no error is shown.
- On failure: the row returns to normal, and the existing error line shows the existing "not
  deleted" message.

## Status announcements

A visually hidden `role="status"` (`aria-live="polite"`) region in the table section, separate
from the error line:

| When | Text |
|------|------|
| Row becomes pending | `{party} on {date} removed. Undo available for 5 seconds.` |
| Undo | `{party} on {date} restored.` |
| Deleted | `{party} on {date} deleted.` |

## Flush event

| Name | Dispatched by | Effect |
|------|---------------|--------|
| `window` `CustomEvent` `hl-budget:flush-deletes` | Settings dialog on open; Import dialog on open | Every waiting entry is deleted at once |

The table also flushes when it unmounts (month, year or tab change). A full page unload sends
nothing.

## Unchanged

- Phone layout (`app/ui/phone/*`): it keeps the delete confirmation sheet and has no quick add.
- Backend endpoints `DELETE /api/v1/spendings/{id}` and `DELETE /api/v1/income/{id}`.
