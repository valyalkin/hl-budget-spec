# UI Contract: Phone Layout

**Feature**: [spec.md](../spec.md) | **Date**: 2026-10-31

This feature exposes no new HTTP interface. The backend's OpenAPI document (v1) is unchanged,
and the frontend's generated types are not regenerated.

The phone layout's contract is its user interface: the roles, accessible names and texts that
users, screen readers and the Playwright phone specs rely on. Names below are exact unless
shown as a pattern.

## Server interfaces used (unchanged)

| Use | Interface | Already used by |
|---|---|---|
| Month data | `loadSpendingsPage` → `/api/v1/users/me`, `/monthly-totals`, `/spendings`, `/category-totals`, `/categories`, `/accounts`, `/merchants` | Desktop (003) |
| Earliest year | `GET /api/v1/spendings?sort=date,asc&size=1&page=0` (new caller of an existing endpoint, R-004) | — |
| Add / edit | Server Action `saveSpending` | Desktop (003) |
| Delete | Server Action `deleteSpending` | Desktop (003) |
| Settings | `createCategory`, `deleteCategory`, `createAccount`, `deleteAccount` | Desktop (003) |
| Import | Server Action `importSpendings` | Desktop (004) |
| Sign in / out | `/auth/login`, `/auth/logout` links | Desktop (001) |

## Layout selection

| Viewport width | Rendered |
|---|---|
| < 640px | Phone layout (after hydration; a phone skeleton before it) |
| ≥ 640px | Desktop layout, unchanged |

The phone tree's root carries `data-layout="phone"` for tests. The desktop tree is not marked
and is not changed.

## Spendings screen

| Element | Role | Accessible name / text |
|---|---|---|
| App name | text | "HL-Budget" (with logo, `aria-hidden` logo) |
| User icon | button, `aria-haspopup="dialog"` | "Account menu" |
| Month | combobox (`<select>`) | "Month"; options "January" … "December"; future months `disabled` |
| Year | combobox (`<select>`) | "Year"; options newest first |
| Month total | text, monospaced | e.g. "$2,800.97" |
| Count | text | "1 spending" / "N spendings" |
| View switch | group "View" with 2 buttons (`aria-pressed`) | "Spendings", "By category" |
| Spending row | button | Named from its content: the visible date, category and amount, then the merchant as visually hidden text, e.g. "Oct 27, Groceries, $64.38, Neighbourhood Market" (spacing around commas varies by browser) |
| Empty list | text | "No spendings in [Month Year]." |
| Floating add | button | "Add spending" |
| Notice | status (`role="status"`) | "Deleted [merchant] · [amount]" / "Added N spendings from [file]" |
| Chart switch | group "Show as" with 2 buttons (`aria-pressed`) | "Table", "Pie" |
| Breakdown table | table "Spending by category" | Columns "Category", "Amount", "Share"; last row "Total", month total, "100%" |
| Pie | img | "Donut chart of [Month Year] spending by category. Values are listed below." |
| Pie legend | list | One item per slice: name, amount, share; "Everything else" when there are more than 6 |
| Empty breakdown | text | "Nothing to break down in [Month Year]." |

## Edit / Add / Fix row screen

A `dialog` named by its title.

| Element | Role | Accessible name / text |
|---|---|---|
| Close | button | "Close without saving" |
| Title | heading level 1 | "New spending" / "Edit spending" / "Fix row" |
| Amount | textbox, `inputmode="decimal"` | "Amount (CAD)" |
| Date | button, `aria-expanded` | "Date" label; shows "Mon D, YYYY" |
| Calendar | group / dialog "Choose date" | "Previous month", "Next month", month title (live), day buttons named "[Weekday], [Month] [D], [YYYY]", selected day `aria-pressed="true"`, future days `disabled`, "Today", "Close" |
| Merchant | combobox (input + listbox) | "Merchant"; placeholder "Where did you spend?" |
| Category | combobox (input + listbox) | "Category"; placeholder "Choose or type" |
| Account | combobox (input + listbox) | "Account"; placeholder "Choose or type" |
| Suggestion | option (button, at least 44px) | the name |
| Error | alert | first problem, e.g. "Enter an amount greater than zero." |
| Delete | button (edit / fix row only) | "Delete spending" / "Remove row" |
| Primary | button, fixed at the bottom | "Add spending" / "Save changes" / "Save row" |

## Bottom sheets

| Sheet | Role / name | Contents |
|---|---|---|
| Account menu | dialog "Account menu" | buttons "Settings", "Import CSV"; link "Sign out"; backdrop button "Close menu" |
| Delete confirmation | alertdialog labelled "Delete this spending?" | text "[merchant] · [amount] · [Mon D, YYYY]"; buttons "Delete spending", "Cancel"; backdrop button "Keep this spending" |

## Settings screen

A `dialog` named "Settings".

| Element | Role | Accessible name / text |
|---|---|---|
| Close | button | "Close settings" |
| Tabs | group "Settings section" with 2 buttons (`aria-pressed`) | "Categories", "Accounts" |
| New category | textbox | "New category"; placeholder "e.g. Coffee, Gifts, Kids" |
| Add | button | "Add category" |
| Own list | heading level 2 + list | "Your categories · N"; empty: "No custom categories yet." |
| Remove | button | "Remove [name]" |
| Defaults | heading level 2 + list | "Default categories · 17"; plain list items, no buttons |
| New account | textbox | "New account"; placeholder "e.g. Mastercard, Savings" |
| Add | button | "Add account" |
| Account item | list item | name; "N spendings recorded" / "1 spending recorded" / "No spendings yet"; button "Remove [name]" |
| Error | alert | desktop messages (003 FR-083, FR-084), e.g. "“Coffee” already exists.", "You need at least one account." |

## Import CSV screens

**Import CSV**: a `dialog` named "Import CSV".

| Element | Role | Accessible name / text |
|---|---|---|
| Close | button | "Close import" |
| Choose file | file input inside a large label-button | "Choose a CSV file"; visible "Choose file"; `accept=".csv,text/csv"` |
| Note | text | "What the file should contain": columns for date, merchant, category, account and amount, with a header row; order doesn't matter |
| Error | alert | 004's messages (not CSV, empty, unreadable, too large, no data rows) |

**Review import**: a `dialog` named "Review import".

| Element | Role | Accessible name / text |
|---|---|---|
| Back | button | "Back to file selection" |
| Subtitle | text | "[file name] · N rows" |
| Banner | status | "N rows need attention" / "1 row needs attention" / "All rows look good" |
| Hint | text | "Tap a row to check or fix it." |
| Row | button | Named from its content: `"[merchant or Merchant missing], [date] · [category] · [account], [amount], needs attention"`; the last part is visually hidden and only there when the row needs it |
| Empty | text | "No rows left to import." |
| Add | button | "Add N spendings" / "Add 1 spending"; `disabled` "Add spendings" while blocked |

## Touch and text guarantees (checked on every phone screen)

- Every visible `button`, `a`, `input`, `select` and `[role=option]` has a bounding box of at
  least 44×44 CSS pixels.
- Every `input` and `select` has a computed `font-size` of at least 16px.
- `document.documentElement.scrollWidth <= clientWidth` at 320, 390 and 430px.
- No information or action is available only on hover.
