# Data Model: Quick add from a date, and undo for delete

No stored data changes. The backend entities (spending, income entry) and the API are
untouched. This file describes the client-side state the feature adds.

## OpenRow (table state, changed)

```text
OpenRow =
  | null
  | { mode: 'add'; date?: string /* YYYY-MM-DD, set by quick add */ }
  | { mode: 'edit'; spendingId: string }
```

- `date` absent: the row behaves as today. The date is today, or the 1st of the shown month,
  and the calendar opens.
- `date` present: the row starts with that date, the calendar closed and focus in merchant.
  The date always belongs to the shown month, because it comes from a listed row.

## Pending delete (hook state, new)

Keyed by entry id. It exists only on the open page.

| Field | Type | Meaning |
|-------|------|---------|
| `id` | string (UUID) | The entry being removed |
| `phase` | `'waiting' \| 'sending'` | Whether the 5 s timer is running, or the delete has been sent |
| timer | handle (ref, not rendered) | Fires 5 000 ms after `start` |

### State transitions

```text
          start(id)                 5 s elapse / flushAll()
 (none) ───────────► waiting ─────────────────────────────► sending
   ▲                    │                                     │
   │   undo(id)         │                                     ├─ ok / gone ─► (none) once the row
   └────────────────────┘                                     │                 leaves the list
   ▲                                                          ├─ error ─────► (none) + message
   └──────────────────────────────────────────────────────────┴─ signed-out ─► navigate to /
 page unload (reload, close, sign out): all timers dropped, nothing sent
```

### Rules

- `start` is ignored if the id is already pending, or if the row is open for editing.
- `undo` is allowed only in `waiting`.
- An id moves to `sending` at most once per page (send-once guard).
- Several ids can be pending at the same time, each with its own timer.
- `flushAll()` moves every `waiting` id to `sending` at once. It is called on unmount and on the
  `hl-budget:flush-deletes` event.
- Totals and breakdowns are computed from server data and ignore pending state. A pending entry
  counts until the server list drops it.

## Row presentation derived from state

| Row state | Date cell | Cells | Actions |
|-----------|-----------|-------|---------|
| Normal, no row open | Button "Add on {date}" | full opacity | Edit, Remove (hover/focus) |
| Normal, another row open | Plain text | full opacity | Edit, Remove disabled (as today) |
| waiting | Plain text | ~50% opacity | Undo, always visible |
| sending | Plain text | ~50% opacity | Undo disabled, always visible |
