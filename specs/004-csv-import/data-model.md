# Data Model: Import Spendings from CSV

**Feature**: [spec.md](./spec.md) | **Research**: [research.md](./research.md)

Spendings, categories and accounts are unchanged from feature 002 (and 003's create-by-name).
This feature adds one small server table and client-only state for the review.

## Server: `spending_imports` (new)

A receipt of one successful import. It exists so a retried import is recognised (idempotency)
and so `201 Created` has a readable `Location` (R-006). It stores nothing from the file.

| Column | Type | Rules |
|---|---|---|
| `id` | `uuid` PK | `gen_random_uuid()` |
| `user_id` | `uuid` NOT NULL | FK `users(id)` ON DELETE CASCADE; every read filters on it |
| `idempotency_key` | `varchar(255)` NULL | unique per user when present |
| `idempotency_fingerprint` | `char(64)` NULL | SHA-256 of the normalised request; present exactly when the key is |
| `spending_count` | `integer` NOT NULL | 1 – 1000 |
| `first_date` | `date` NOT NULL | earliest spending date in the import |
| `last_date` | `date` NOT NULL | latest; `first_date <= last_date` |
| `total` | `numeric(14,2)` NOT NULL | sum of the amounts, > 0; 14 digits hold 1,000 × the per-spending maximum 99,999,999.99 |
| `created_at` | `timestamptz` NOT NULL | `now()` |

Migration `V4__create_spending_imports.sql` (additive):

- `CONSTRAINT ck_spending_imports_count CHECK (spending_count BETWEEN 1 AND 1000)`
- `CONSTRAINT ck_spending_imports_dates CHECK (first_date <= last_date)`
- `CONSTRAINT ck_spending_imports_idempotency_pair CHECK ((idempotency_key IS NULL) = (idempotency_fingerprint IS NULL))`
- `CREATE UNIQUE INDEX ux_spending_imports_user_key ON spending_imports (user_id, idempotency_key) WHERE idempotency_key IS NOT NULL`

No column links spendings to their import (R-006): spendings remain ordinary and individually
editable and deletable.

### Fingerprint

SHA-256 over the rows in request order. Each row is normalised as
`date|trim(merchant)|amount at scale 2|currency|lower(trim(categoryName))|lower(trim(accountName))`,
rows joined with `\n`. Names are lower-cased, so a retry that typed "visa" instead of "Visa" still
counts as the same request, matching how names are resolved.

### Write: create an import (one transaction)

1. Validate every row (Bean Validation, then `SpendingRules` with the row's pointer prefix).
   Any violation → `422` with all of them; nothing written (FR-033).
2. With a key: if the caller already has an import with that key → same fingerprint: return it
   (`200`, `Idempotent-Replayed: true`); different fingerprint: `422`
   `idempotency-key-reused`.
3. Insert the `spending_imports` row with `ON CONFLICT (user_id, idempotency_key) DO NOTHING`. If
   another request with the same key won the race, read the winner and go to step 2's replay.
4. Resolve names: for each distinct `lower(trim(name))` among categories, then accounts, call the
   existing `NamedItemInserts.findOrCreate…` once (FR-032: one item per name ignoring case).
5. Batch-insert all spendings (JDBC batch), with no idempotency key of their own.
6. Commit. Any failure rolls back the import row, new names and spendings together.

### Read: one import

`GET /api/v1/spending-imports/{id}` → the row as `SpendingImport` when `user_id` is the caller's,
else `404` (no existence leak).

## Client state (frontend only, never stored)

### Import session (`ImportProvider`)

| Field | Type | Notes |
|---|---|---|
| `stage` | `'closed' \| 'pick' \| 'review'` | |
| `fileName` | `string` | shown in the review header and the "Added …" notice |
| `rows` | `ImportRow[]` | file order |
| `pickError` | `string \| null` | refusal message in the pick stage (R-012) |
| `idempotencyKey` | `string` (UUID) | new when the review opens, and after the first edit following a `422` with row errors; kept after "unreachable" or a 5xx (R-013) |
| `adding` | `boolean` | disables the add button while the request runs (FR-034) |
| `serverErrors` | `Map<rowKey, Partial<Record<Field, string>>>` | from a `422`; cleared for a field when the user edits it |
| `notice` | `string \| null` | "Added N spendings from [file]" after success |
| `lookups` | `{ categories, accounts, merchants, today, currency }` | passed by `ImportCsvButton` on open |

### `ImportRow`

| Field | Type | Notes |
|---|---|---|
| `key` | `string` | stable React key (row number at parse time); not the position, since rows can be removed |
| `draft` | `Draft` | feature 003's type: `date`, `merchant`, `category`, `account`, `amount`, all strings |

Derived on every render (not stored): `problems = draftProblems(draft, today)` merged with
`serverErrors[key]`; the row is valid when both are empty.

### Derived review figures

- **Rows needing attention**: count of rows with any problem → badge "N rows need attention" or
  "All rows look good" (FR-020).
- **Total**: sum in integer cents (`toCents`) of amounts that parse, shown with `formatCents`.
- **Can add**: `rows.length > 0 && every row valid && !adding` (FR-031).

### Parsing pipeline (file → rows)

`File` → size/type checks → `text()` → `parseCsv` (R-002) → `mapColumns` (R-003) → per row:
`parseImportDate` (R-004), `normaliseName` for merchant/category/account (trim, collapse spaces),
`exactMatch` against the user's names (FR-014), `parseImportAmount` (R-005) → `ImportRow[]`.
Values that cannot be read become `''` (FR-015).
