# Data model: Income (feature 009)

Decisions are in [research.md](./research.md) (R-001, R-002). Everything is added by one Flyway
migration, `V7__create_incomes.sql`; V1 to V6 are untouched.

## Tables

### `users` (changed)

| Column | Type | Notes |
|---|---|---|
| `income_categories_seeded_at` | `timestamptz NULL` | **New.** When Salary and Investments were created for this user. NULL until their first budgeting call after this feature, including existing users (R-002) |

### `income_categories` (new)

Mirrors `categories` (V3 + V5).

| Column | Type | Rules |
|---|---|---|
| `id` | `uuid` PK | `gen_random_uuid()` |
| `user_id` | `uuid NOT NULL` | FK `users(id) ON DELETE CASCADE` |
| `name` | `varchar(50) NOT NULL` | 1 to 50 characters, already trimmed (`ck_income_categories_name`) |
| `is_default` | `boolean NOT NULL DEFAULT false` | true for Salary and Investments |
| `color` | `varchar(7) NOT NULL` | `^#[0-9a-f]{6}$` (`ck_income_categories_color`); no default: the app always sets it |
| `created_at`, `updated_at` | `timestamptz NOT NULL DEFAULT now()` | |
| `version` | `bigint NOT NULL DEFAULT 0` | optimistic lock |

- `uq_income_categories_id_user UNIQUE (id, user_id)`: target of the composite FK from `incomes`.
- `ux_income_categories_user_lower_name UNIQUE (user_id, lower(name))`: unique per user ignoring
  case, defaults included, independent of `categories` (FR-022).

### `incomes` (new)

Mirrors `spendings` (V3).

| Column | Type | Rules |
|---|---|---|
| `id` | `uuid` PK | |
| `user_id` | `uuid NOT NULL` | FK `users(id) ON DELETE CASCADE` |
| `income_date` | `date NOT NULL` | not after today in the app's time zone (checked in `EntryRules`) |
| `source` | `varchar(255) NOT NULL` | 1 to 255 characters, already trimmed (`ck_incomes_source`) |
| `amount` | `numeric(10,2) NOT NULL` | `> 0` (`ck_incomes_amount`) |
| `category_id` | `uuid NOT NULL` | FK `(category_id, user_id) → income_categories(id, user_id) ON DELETE NO ACTION` |
| `account_id` | `uuid NOT NULL` | FK `(account_id, user_id) → accounts(id, user_id) ON DELETE NO ACTION` |
| `idempotency_key` | `varchar(255) NULL` | with `idempotency_fingerprint char(64) NULL`; both or neither (`ck_incomes_idempotency_pair`) |
| `created_at`, `updated_at` | `timestamptz NOT NULL DEFAULT now()` | |
| `version` | `bigint NOT NULL DEFAULT 0` | |

Indexes (mirroring `spendings`, R-010):
- `ux_incomes_user_idempotency_key (user_id, idempotency_key) WHERE idempotency_key IS NOT NULL`
- `ix_incomes_user_date (user_id, income_date DESC, created_at DESC, id)`: list order, month
  totals, sources
- `ix_incomes_user_category (user_id, category_id)`: filters, counts, in-use check
- `ix_incomes_user_account (user_id, account_id)`: filters, account `incomeCount`, in-use check

### `income_imports` (new)

Mirrors `spending_imports` (V4): `id`, `user_id`, `idempotency_key` + `idempotency_fingerprint`,
`income_count integer` (1 to 1000), `first_date`, `last_date` (`first_date <= last_date`),
`total numeric(14,2) > 0`, `created_at`. Unique `(user_id, idempotency_key)` where the key is set.
Stores nothing from the file.

### `accounts` (unchanged)

No column changes. The FK from `incomes` means an account used by income can't be deleted at the
database level either (FR-052).

## Entities and rules

| Entity | Fields (API) | Rules | Spec |
|---|---|---|---|
| **Income** | `id`, `date`, `source`, `amount` (Money), `category` (Ref), `account` (Ref), `createdAt`, `updatedAt` | date ≤ today; source 1–255 after trim; amount > 0, ≤ 2 decimals, app currency; category one of the caller's income categories, account one of the caller's accounts, each by id or by name, never created | FR-010–FR-013 |
| **IncomeCategory** | `id`, `name`, `isDefault`, `incomeCount` (derived), `color` | name 1–50 after trim, unique among the caller's income categories ignoring case; defaults not renamed or deleted; delete refused while `incomeCount > 0` | FR-020–FR-025 |
| **Default income categories** | Salary `#1e5b45`, Investments `#1d4ed8` | seeded once per user, never duplicated | FR-020, FR-024 |
| **Account** (changed) | + `incomeCount` (derived) | delete refused while `spendingCount + incomeCount > 0`; last-account rule unchanged | FR-050–FR-053 |
| **IncomeSource** (derived) | `name`, `incomeCount`, `lastUsedOn`, `lastCategory`, `lastAccount` | incomes grouped by `lower(btrim(source))`; name, category and account from the most recent entry; most used first | FR-063 |
| **Income month totals** (derived) | per month: `total`, `entryCount` | sums of exact `numeric` in Postgres | FR-065 |
| **Income category / account totals** (derived) | per item: `category` or `account` (Ref), `total`, `entryCount`, `color` | only items used in the range, largest first, ties by name | FR-064 |
| **IncomeImport** | `id`, `incomeCount`, `firstDate`, `lastDate`, `total`, `createdAt` | 1–1000 rows, all or nothing, retry-safe by `Idempotency-Key` | FR-080, FR-081 |

## Colour assignment (R-002)

- Salary and Investments: fixed, by name, the same for everyone.
- A new custom income category: `CategoryPalette.next(counts)`, where `counts` is the number of the
  caller's income categories per colour. Spare colours first (`#e11d48`, `#0891b2`, …), then the
  spending defaults' colours, so the first 22 custom income categories each get an unused colour.
- Kept on rename; a removed category frees its colour.

## State and lifecycle

No status fields. An income is created, replaced in full (PUT) or deleted (hard delete). An income
category is created in Settings, renamed (API only; Settings offers add and remove) or deleted when
unused. Seeding is a one-way `NULL → timestamp` transition on `users.income_categories_seeded_at`,
claimed with a conditional update so concurrent first calls seed once.

## Frontend view types

- `LedgerEntry` (UI): the existing spending row shape; for income, `merchant` holds the source
  (R-006). Mapped in `data.ts` (read) and `actions.ts` (write).
- `NamedItem` lists for the entry row and Settings: income categories with `count = incomeCount`.
- Breakdown items: unchanged `BreakdownItem` (`id`, `name`, `cents`, `color`).
