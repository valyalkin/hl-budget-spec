# Research: Import Spendings from CSV

**Feature**: [spec.md](./spec.md) | **Plan**: [plan.md](./plan.md) | **Date**: 2026-10-29

Each decision has an id (R-001 …) that the plan, data model, contract and tasks refer to.

## R-001 Where the file is read: in the browser

**Decision**: The browser reads and parses the file. The file is never uploaded. Only the rows
the user confirms reach the server, as one JSON request (R-006).

**Rationale**:
- FR-051: the file's contents must not be stored or logged anywhere. If it never leaves the
  browser, it cannot be.
- SC-005: a 1,000-row file opens in the review in under 3 s. Local parsing takes milliseconds,
  with no upload round trip.
- Reading a file into draft rows is input handling, like turning keystrokes into a draft in
  feature 003's entry row. The business rules (valid date, positive amount, name matching and
  creation, all-or-nothing) stay in the backend, which re-checks every row (Principle II). The
  browser's checks only mirror the server's, as `draft.ts` already does.
- No new upload endpoint, multipart handling or file-size settings on the backend or ingress.

**Alternatives considered**:
- *Upload the file to a backend "preview" endpoint that parses it*: puts parsing next to the
  rules, but sends the raw file over the network, adds an endpoint, multipart limits and a
  second round trip, and the review needs the rows as editable client state anyway.
- *Parse in a Server Action*: the file still travels to a server, and Server Action bodies are
  limited to 1 MB by default, the same as our file limit, which leaves no headroom.

## R-002 CSV parser: in-house, no dependency

**Decision**: A small RFC 4180 parser in `app/lib/csv.ts` (about 40 lines, as in the design's
prototype). It handles quoted values with commas, `""` escapes and line breaks; `\n`, `\r\n` and
`\r` line endings; skips blank lines; strips a leading byte-order mark. Comma only.

**Rationale**: The format needed (FR-012) is small and fully testable with unit tests. Feature
003 set the precedent of building small pieces in-house (combobox, calendar, donut) instead of
adding runtime dependencies (Principle VI).

**Alternatives considered**: *Papa Parse*: robust, delimiter sniffing and streaming, but those
features are out of scope (comma-only, 1 MB), and it adds a runtime dependency for ~40 lines.

## R-003 Recognising columns

**Decision**: `app/lib/csv-import.ts` maps the header row to the five fields. Header cells are
trimmed and lower-cased, then each field takes the first unused column whose name matches its
pattern, checked in this order:

| Field | Pattern (substring) |
|---|---|
| date | `date`, `posted`, `day` |
| category | `categ` |
| account | `account`, `card`, `source` |
| amount | `amount`, `total`, `value`, `debit`, `price`, `cost` |
| merchant | `merchant`, `description`, `payee`, `vendor`, `name`, `memo`, `store`, `details` |

Merchant comes last so that a header like "Account name" goes to account, not merchant. When
fewer than two fields are recognised, the file has no header: columns are read in the fixed
order date, merchant, category, account, amount, and row 1 is data (spec Edge Cases).

**Rationale**: This is the design prototype's rule, written out so it can be tested. Order
matters, so it is a list, not a map.

**Alternatives considered**: *A column-mapping step*: out of scope by the spec's assumptions.

## R-004 Reading dates: year first or day first only

**Decision**: `parseImportDate(text)` returns `YYYY-MM-DD` or `''`:
- `^(\d{4})[-/.](\d{1,2})[-/.](\d{1,2})$` → year, month, day.
- `^(\d{1,2})[-/.](\d{1,2})[-/.](\d{4})$` → **day, month, year**, always (clarification Q2).
- Anything else, or an impossible date (month 13, 30 February), → `''`.

Time parts after the date (`2026-10-28 14:03`, `2026-10-28T14:03:00`) are cut at the first space
or `T` before matching, because bank exports often include them. Future dates are kept as read
and flagged by validation (R-008), so the user sees what the file said.

**Rationale**: Clarification Q2 chose day first, matching Canadian and Amex exports. Leaving
unreadable dates empty, instead of refusing the file, is FR-015.

**Alternatives considered**: The design's "month first unless the first number is over 12"
(rejected by Q2); inferring the order from the whole file (rejected by Q2).

## R-005 Reading amounts without floating point

**Decision**: `parseImportAmount(text)` returns a canonical decimal string (`"18.90"`) or `''`:
1. Remove spaces, currency symbols (`$`, `€`, `£`, `¥`, and three-letter codes such as `CAD`) and
   thousands separators (`,`).
2. Treat `(x)` and a leading or trailing `-` as negative, and drop the sign (spec: negative
   amounts are spendings).
3. Accept `^\d+(\.\d+)?$` only; round half up to 2 decimals on the decimal string, never through
   a JavaScript number (constitution: no float money). Reuse `toCents`/`centsToAmount` from
   `utils.ts` for the rounding.
4. Zero, or anything else, → `''`.

The resulting string then goes through feature 003's `parseAmountInput`, so the review shows and
validates amounts exactly as the entry row does.

**Rationale**: The design prototype used `parseFloat` and `Math.abs`. That is fine for a mockup
but breaks the constitution's money rule.

## R-006 Adding many spendings at once: a `spending-imports` resource

**Decision**: A new operation `POST /api/v1/spending-imports` creates all the rows in one
transaction and records the import. Its body is `{ "spendings": [SpendingImportRow, …] }`
(1 to 1,000 rows). Each row has `date`, `merchant`, `amount` (Money), `categoryName` and
`accountName`, all required. Responses:
- `201` with `Location: /api/v1/spending-imports/{id}` and a `SpendingImport` body:
  `id`, `spendingCount`, `firstDate`, `lastDate`, `total` (Money), `createdAt`.
- `200` with `Idempotent-Replayed: true` when the same `Idempotency-Key` and body were already
  imported. Nothing new is created.
- `422` with every invalid field, its JSON Pointer naming the row (R-007).

`GET /api/v1/spending-imports/{id}` returns the same summary. It exists because a `201` must
point at a readable resource (Principle VII).

A new table `spending_imports` (data-model.md) holds that summary and the idempotency key. It
stores no file name, no row contents and no link to the spendings. Spendings stay ordinary
spendings: editable and deletable one by one, as the spec requires.

**Rationale**:
- FR-033 requires all-or-nothing. N calls to `POST /spendings` cannot give that, and would take
  1,000 round trips for a full file (SC-001).
- Principle VII: URLs are resources, not actions, and a `201` comes with a `Location`. An import
  is a resource that is created; `/spendings/batch` or `/spendings:import` would be an action.
- Idempotency (FR-034, Principle VII SHOULD): a retried import must not add everything twice.
  Spendings hold one idempotency key each, so the batch needs its own key. A row in
  `spending_imports` with a unique `(user_id, idempotency_key)` gives the batch exactly the
  race-safe `INSERT … ON CONFLICT DO NOTHING` pattern feature 002 uses for spendings (R-007 of
  002).
- Names only, no ids: the importer always has names from the file. The server already matches
  names case-insensitively and creates missing ones (feature 003, R-004), so the frontend does
  not need to look up ids first.

**Spec note**: the spec's assumption "No saved import history" still holds for the user. The
`spending_imports` row is an internal receipt (counts, dates, total, key), never listed or shown,
and it keeps nothing from the file (FR-051).

**Alternatives considered**:
- *`POST /spendings` once per row from the Server Action*: not atomic; 1,000 calls.
- *Per-row derived idempotency keys (`<key>/<n>`) on spendings, no new table*: avoids a
  migration, but a replay has to find and check up to 1,000 rows, deleting one spending frees its
  key and breaks the replay, and there is still no resource for `Location`.
- *`spendings.import_id` column linking spendings to their import*: would allow "undo import"
  later, but the spec puts undo out of scope and Principle VI says not to add it until needed.
  It can be added later as a nullable column without breaking anything.

## R-007 Validating many rows and reporting by row

**Decision**:
- Bean Validation on `SpendingImportRequest`: `@Size(min = 1, max = 1000)` on `spendings`; each
  row is `@Valid` with the same annotations as `SpendingRequest` (merchant not blank, at most
  255; names not blank, at most 50; Money valid). Violations already come out as JSON Pointers
  like `#/spendings/3/merchant`.
- Rules that need the clock or configuration (date not in the future, amount positive, currency)
  come from `SpendingRules.violations(...)`, which gains a pointer prefix (`#/spendings/3`), so
  one row reports `#/spendings/3/date`.
- All violations of all rows are collected and returned together in one `422`
  (`SpendingValidationFailed`, reason `validation`). Nothing is created unless every row is
  valid.

On the frontend, `importSpendings` maps each `#/spendings/{i}/{field}` back to row `i` and field
(`amount/amount` → amount, `categoryName` → category, `accountName` → account), and the review
highlights those fields with the server's message (spec Edge Case "Adding fails").

**Rationale**: FR-022 and the "Adding fails" edge case need to know which row and field the
server refused. JSON Pointers into the request body are the constitution's format.

## R-008 Validation in the review

**Decision**: `draft.ts` gains `draftProblems(draft, today)`, which returns every problem of a
draft (not only the first) as a field → message map. The existing `draftProblem` becomes "the
first of `draftProblems`", so feature 003's behaviour is unchanged. The review computes problems
for every row on each render. That is cheap: 1,000 rows × 5 string checks.

Messages for highlighted fields reuse `MESSAGES` (`Pick a date.`, `Pick a date that is today or
earlier.`, …). Each invalid field gets `aria-invalid="true"` and `aria-describedby` pointing at
its message, shown as a tooltip-style line under the row only while the field has focus, to keep
the table compact (FR-022, FR-052).

## R-009 Review table: reuse the entry row's parts, not the entry row

**Decision**: A new `ImportRow` component renders one row of the review with feature 003's
`DatePicker` and `Combobox` (merchant, category, account) and a plain amount input, plus a
remove button. Keyboard handling shared with `EntryRow` (Left/Right arrow stepping, Enter to the
next field, the merchant fill-in of an empty category and account) moves out of `entry-row.tsx`
into `app/ui/spendings/row-keys.ts` and is used by both.

Differences from the entry row (FR-023):
- A row never saves. Enter on the amount field moves to the next row's date; Tab works natively
  through the whole table.
- Escape closes an open list or calendar; otherwise it goes to the dialog (FR-041).
- Autofill data (categories, accounts, merchants with counts) is the same as the Spendings
  table's, passed in when the dialog opens.

**Rationale**: `EntryRow` is built around one row that saves itself (Tab on amount saves,
Escape cancels). Changing it into a dual-purpose component would add mode switches throughout.
The reusable parts are the inputs and the key helpers.

**Performance**: 1,000 rows render about 5,000 inputs. Comboboxes render their list only while
open and the calendar only while open, so the table is plain inputs. A quickstart check measures
SC-005 with a 1,000-row file. If it misses, rows outside the scroll box are rendered with
`content-visibility: auto` before considering virtualisation.

## R-010 The dialog, the button and the "Added" message

**Decision**:
- `ImportProvider` (client) wraps the Spendings page's content in `app/spendings/page.tsx`,
  **outside** the `<Suspense>` boundary, so it keeps its state when the month changes. It holds
  the dialog stage (`closed | pick | review`), the rows, the file name and the last notice.
- `ImportCsvButton` (client) sits in the month summary `aside` above `MonthTotalCard` (FR-001).
  It calls the provider's `open({ categories, accounts, merchants, today, currency })` with the
  page's data, so the provider never fetches.
- `ImportDialog` is a native `<dialog>` opened with `showModal()`, like the Settings dialog.
  Focus stays inside, the page behind is inert, and Escape fires `cancel` (FR-002). In the review
  stage, `cancel` is intercepted: it closes an open popup first, otherwise returns to the pick
  stage (FR-041).
- After a successful import: the dialog closes, the router goes to
  `/spendings?month=<latest imported month>` (FR-035), and the provider shows
  "Added N spendings from [file name]" in a `role="status"` line at the top of the page. It
  stays, across month changes, until the user dismisses it or opens Import CSV again (as in the
  design).
- Opening the import closes an open entry row (spec Edge Case): `ImportCsvButton` is disabled
  while a row is open, like the + button. Simpler than closing someone's row for them, and
  consistent with feature 003's one-thing-at-a-time rule.

**Rationale**: The notice must survive the navigation to another month. Anything inside the
`<Suspense>` boundary may remount on a `searchParams` change. Passing it through the URL would
put the file name in the address bar and show it again on reload.

**Spec note**: the last bullet differs slightly from the spec's edge case ("closes that row
without saving"). Disabling the button keeps the user's half-typed row safe, and the spec's goal
(never two edits at once) holds. Recorded here; the spec edge case is updated to match.

## R-011 The sample file

**Decision**: A static file `public/sample-spendings.csv`, linked with
`<a href="/sample-spendings.csv" download>Download sample CSV</a>` (FR-005). Contents:

```csv
Date,Merchant,Category,Account,Amount
28/10/2026,Corner Bakery,Groceries,Visa,12.40
28/10/2026,Transit fare,Public Transport,Chequing,3.35
2026-10-27,Harbour Coffee,Coffee,Visa,6.75
27/10/2026,Dr. Patel Dental,,Visa,185.00
26/10/2026,Birch Yoga Studio,Sport,,24.00
25/10/2026,"Lakeshore Fuel, Hwy 7",Car,Visa,61.20
24/10/2026,Thrift & Co.,Clothes,Visa,
23/10/2026,Ride share,Taxi,Visa,-18.90
22/10/2026,City Library,Education,Cash,5.00
03/10/2026,Hardware Depot,Household,Cash,"1,041.18"
```

Ten rows: both date forms, a missing category, account and amount, a negative amount, a quoted
merchant with a comma, a thousands separator, and a day-first date whose day is 12 or less.
The dates are fixed in the past, so the sample stays valid. A unit test parses this exact file
and checks every row, so the file and the parser cannot drift apart.

**Rationale**: Clarification Q1 chose download only. A static file needs no code, works with the
existing CSP (`'self'`), and can be opened in any spreadsheet.

## R-012 Reading the file in the browser

**Decision**: In `ImportDialog`:
1. Take the first file of a drop or of the file input (spec Edge Case "Several files").
2. Refuse when the name does not end in `.csv` and the type is not `text/csv` or
   `application/vnd.ms-excel` (some systems label CSV so): "That doesn't look like a CSV file.
   Choose a .csv export."
3. Refuse when `file.size > 1 MB`, before reading: "That file is too large. Import at most 1 MB
   and 1,000 rows at a time."
4. `await file.text()` (UTF-8). On failure: "The file couldn't be read. Try again."
5. Parse (R-002 to R-005). More than 1,000 data rows: the same too-large message. No data rows:
   "No spendings found in that file. Check that it is a CSV with a header row and at least one
   data row."

Category and account values are matched to the user's existing names ignoring case
(`exactMatch` from `utils.ts`) and shown with the existing spelling (FR-014).

**Rationale**: `File.text()` is supported by all evergreen browsers and avoids `FileReader`
callbacks. The size check comes first so a huge file is never read into memory.

## R-013 Server Action `importSpendings`

**Decision**: A new action in `app/lib/actions.ts`:
`importSpendings({ rows, idempotencyKey, today, currency })`.
- Validates its input with Zod (at most 1,000 rows, a UUID key) and every row with
  `draftProblems`. Any problem returns `{ status: 'error', rowErrors }` without calling the API.
- Makes one call: `POST /api/v1/spending-imports` with the `Idempotency-Key` header.
- On success: `refresh()`, then returns
  `{ status: 'ok', month: monthParam(monthOf(lastDate)), count: spendingCount }`.
- On `422`: returns `rowErrors: { row, field, message }[]` from the problem's `errors` (R-007).
- Lost session: `signed-out`, as in the other actions (spec Edge Case "Session expiry").

The key is created once when the review opens (`crypto.randomUUID()`) and kept across retries,
so a double click or a retry after a timeout imports once (FR-034). When the key changes
depends on why the add failed:
- **A `422` with row errors**: the server refused and saved nothing, so the first edit
  afterwards makes a new key.
- **Unreachable or a 5xx**: the server may have committed before the answer was lost, so the
  key is **kept**, even if the user edits rows. If the first attempt did commit, the retry's
  different body gets `422 idempotency-key-reused`. The action turns that into
  `{ status: 'already-imported' }`, and the review closes with "These spendings were already
  added." instead of importing a second time. If it did not commit, the retry simply creates
  the import.

**Body size**: 1,000 rows of about 200 bytes of JSON each is well under Next.js's 1 MB Server
Action body limit, so `serverActions.bodySizeLimit` stays at its default.

## R-014 Tests

**Backend** (`./gradlew check`, Kover floors unchanged):
- Unit: `SpendingImportService` (all-or-nothing, names created once per name ignoring case,
  pointers per row, replay and key reuse), and `SpendingRules` with a pointer prefix.
- ITs on Testcontainers Postgres:
  - `SpendingImportsIT`: 201 + Location; GET by id; 422 with row pointers and nothing created;
    replay 200; key reused with another body 422; 1,001 rows 422; 401.
  - `SpendingImportOwnershipIT`: another user's import id is 404; names resolve only within the
    caller's own items.
  - `SpendingImportPerformanceIT`: 1,000 rows import in under 2 s.

**Frontend** (`pnpm typecheck && pnpm lint && pnpm test`):
- Unit: `csv.ts` (quoting, line endings, BOM, blank lines); `csv-import.ts` (header recognition,
  no-header fallback, dates, amounts, name matching, the sample file); `draftProblems`;
  `importSpendings` with MSW (success, 422 mapping, signed-out, unreachable).
- Components: `ImportDialog` (pick, drag highlight, every refusal message, review, cancel back,
  Escape order), `ImportRow` (highlighting, autofill, keys, remove), `ImportCsvButton`.
- Playwright: `e2e/import-csv.spec.ts` covers US1, US2 and US4. `e2e/a11y.spec.ts` gains axe
  scans of both dialog stages. US3 is a download check in the same spec.

## R-015 Branches and delivery

**Decision**: Create `004-csv-import` from `origin/main` in both service repos. The backend PR
merges and deploys first. The new operations are unused by the current frontend, so this is
safe. The frontend then regenerates its types from `contracts/api.yaml` and ships. The Flyway
migration `V4__create_spending_imports.sql` only adds a table.
