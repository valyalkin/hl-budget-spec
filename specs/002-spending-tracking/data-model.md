# Data Model: Spending Tracking (Backend)

**Feature**: `002-spending-tracking` | **Date**: 2026-10-27 | **Plan**: [plan.md](./plan.md)

Only `hl-budget-backend` stores data (Principle II). This feature adds three tables and one
column, in two Flyway migrations, both additive (Principle V, 001's `V1` is untouched):

- `V2__add_users_categories_seeded_at.sql`: the seeding flag (R-003)
- `V3__create_accounts_categories_spendings.sql`: the three tables, constraints and indexes

```text
users 1 ──< accounts    (user_id)
users 1 ──< categories  (user_id)
users 1 ──< spendings   (user_id)
accounts   1 ──< spendings  (account_id, user_id)   ON DELETE NO ACTION
categories 1 ──< spendings  (category_id, user_id)  ON DELETE NO ACTION
```

The spending → account/category FKs are **composite with `user_id`**. A spending can therefore
reference only an account or category of its own user, whatever the application does (FR-031,
SC-003).

`NO ACTION` refuses a delete just like `RESTRICT`. It is used instead because its check runs at
the end of the statement, so a future user delete can still cascade to that user's spendings,
accounts and categories in one statement.

---

## User (existing, extended)

| Column | Type | Null | Default | Notes |
|---|---|---|---|---|
| `categories_seeded_at` | `timestamptz` | yes | — | **New.** Set once, when the 17 defaults are inserted (R-003). Null means not seeded yet |

`UserEntity` gains the field; the 001 upsert does not touch it. Users from before this feature
have null and are seeded on their first budgeting call.

## Account

### Table `accounts`

| Column | Type | Null | Default | Notes |
|---|---|---|---|---|
| `id` | `uuid` | no | `gen_random_uuid()` | PK; the API `id` |
| `user_id` | `uuid` | no | — | FK → `users(id)` `ON DELETE CASCADE` |
| `name` | `varchar(50)` | no | — | Trimmed, 1–50 characters |
| `created_at` | `timestamptz` | no | `now()` | |
| `updated_at` | `timestamptz` | no | `now()` | Set by the service on rename |
| `version` | `bigint` | no | `0` | `@Version` |

Constraints and indexes:

- `pk_accounts` on `id`; `uq_accounts_id_user` unique `(id, user_id)` (the target of the
  composite FK).
- `fk_accounts_user` → `users(id)` on delete cascade. No user delete exists yet; cascade only
  documents ownership.
- `ux_accounts_user_lower_name` unique `(user_id, lower(name))` → FR-022.
- `ck_accounts_name` `length(name) BETWEEN 1 AND 50 AND name = btrim(name)`.

### Domain model `model/Account`

`id: UUID`, `name: String`, `spendingCount: Long` (derived, R-011).

## Category

### Table `categories`

Same columns, constraints and indexes as `accounts` (`pk_categories`, `uq_categories_id_user`,
`fk_categories_user`, `ux_categories_user_lower_name`, `ck_categories_name`), plus:

| Column | Type | Null | Default | Notes |
|---|---|---|---|---|
| `is_default` | `boolean` | no | `false` | True only for the 17 rows inserted by seeding. Never changed by the API |

### Domain model `model/Category`

`id: UUID`, `name: String`, `isDefault: Boolean`, `spendingCount: Long` (derived).

### Default list `model/DefaultCategories`

`Rent, Groceries, Public Transport, Car, Household, Health, Beauty, Clothes, Education, Travel,
Bills, Subscriptions, Other, Sport, Entertainment, Pet, Taxi`. That is 17 names, in English,
inserted exactly as written.

## Spending

### Table `spendings`

| Column | Type | Null | Default | Notes |
|---|---|---|---|---|
| `id` | `uuid` | no | `gen_random_uuid()` | PK; the API `id` |
| `user_id` | `uuid` | no | — | FK → `users(id)` on delete cascade |
| `spending_date` | `date` | no | — | API `date`. Not after "today" (checked in the service; it depends on the clock and time zone) |
| `merchant` | `varchar(255)` | no | — | Trimmed, 1–255 |
| `amount` | `numeric(10,2)` | no | — | > 0, ≤ 99,999,999.99. Currency is the configured one (R-006), not stored |
| `category_id` | `uuid` | no | — | Composite FK `(category_id, user_id)` → `categories(id, user_id)` `ON DELETE NO ACTION` |
| `account_id` | `uuid` | no | — | Composite FK `(account_id, user_id)` → `accounts(id, user_id)` `ON DELETE NO ACTION` |
| `idempotency_key` | `varchar(255)` | yes | — | R-007 |
| `idempotency_fingerprint` | `char(64)` | yes | — | SHA-256 hex of the normalised create command; set exactly when `idempotency_key` is set |
| `created_at` | `timestamptz` | no | `now()` | FR-035; also the paging tie-break |
| `updated_at` | `timestamptz` | no | `now()` | FR-035; set by the service on every update |
| `version` | `bigint` | no | `0` | `@Version` |

Constraints and indexes:

- `pk_spendings` on `id`.
- `ck_spendings_amount` `amount > 0`; `ck_spendings_merchant`
  `length(merchant) BETWEEN 1 AND 255 AND merchant = btrim(merchant)`.
- `ck_spendings_idempotency_pair`
  `(idempotency_key IS NULL) = (idempotency_fingerprint IS NULL)`.
- `ux_spendings_user_idempotency_key` unique `(user_id, idempotency_key)` where
  `idempotency_key IS NOT NULL`.
- `ix_spendings_user_date` `(user_id, spending_date DESC, created_at DESC, id)`: default list
  order (R-010).
- `ix_spendings_user_category` `(user_id, category_id)`;
  `ix_spendings_user_account` `(user_id, account_id)`: filters, counts and the in-use check.

### Domain model `model/Spending`

`id`, `date: LocalDate`, `merchant: String`, `amount: Money`, `category: Ref`, `account: Ref`,
`createdAt: Instant`, `updatedAt: Instant`.

- `Money(amount: BigDecimal, currency: Currency)`. The scale is always 2 when rendered
  (`"84.30"`, never `"84.3"`).
- `Ref(id: UUID, name: String)`.

Commands (service inputs, built from DTOs in `dto/`):

- `SpendingCommand(date, merchant, amount: BigDecimal, categoryId, accountId)`
- `NameCommand(name)`

## Validation rules

| Rule | Where | On failure |
|---|---|---|
| Body is JSON and the path id is a UUID | Jackson / Spring | 400 `bad-request` |
| Field present and of the right JSON type | Jackson | 422, `errors[]` code `Required` / `TypeMismatch` |
| `name` not blank, ≤ 50 after trim | DTO + DB check | 422 `NotBlank` / `Size` |
| `merchant` not blank, ≤ 255 after trim | DTO + DB check | 422 `NotBlank` / `Size` |
| `amount.amount` matches `^\d{1,8}(\.\d{1,2})?$` | DTO | 422 `Pattern` at `#/amount/amount` |
| `amount.amount` > 0 | Service | 422 `Positive` at `#/amount/amount` |
| `amount.currency` equals the configured currency | Service | 422 `UnsupportedCurrency` at `#/amount/currency` |
| `date` ≤ today in `hlbudget.time-zone` | Service | 422 `NotFuture` at `#/date` |
| `categoryId` / `accountId` exists and belongs to the caller | Service + composite FK | 422 `NotFound` at `#/categoryId` / `#/accountId` |
| Name unique per user, ignoring case | Service pre-check + unique index | 409 `reason: duplicate-name` |
| Default category renamed or deleted | Service | 422 `reason: default-category` |
| Account or category in use deleted | Service count + FK `NO ACTION` | 409 `reason: in-use`, `spendingCount` |
| `Idempotency-Key` 1–255 visible ASCII | Controller | 422 at the header |
| Same key, different body | Service | 422 `reason: idempotency-key-reused` |
| Path id unknown or another user's | Service | 404 `not-found` (same response either way) |
| Concurrent write lost | `@Version` | 409 `reason: concurrent-modification` (existing handler) |
| List: `size` 1–100, `page` ≥ 0, `from` ≤ `to`, `sort` in the allowed set | Controller / service | 422 at the parameter |

## Lifecycle

```text
User:      categories_seeded_at NULL ──first budgeting call──▶ set (17 defaults inserted, once)
Account:   (none) ─POST─▶ ACTIVE ─PUT─▶ ACTIVE ─DELETE (0 spendings)─▶ gone
                                         └─DELETE (n > 0)─▶ 409 in-use, unchanged
Category:  custom: same as Account
           default: created by seeding; PUT/DELETE ─▶ 422 default-category, unchanged
Spending:  (none) ─POST─▶ STORED ─PUT─▶ STORED ─DELETE─▶ gone (hard delete; key freed)
```
