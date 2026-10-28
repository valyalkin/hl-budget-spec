# Research: Spending Tracking (Backend)

**Feature**: `002-spending-tracking` | **Date**: 2026-10-27 | **Plan**: [plan.md](./plan.md)

Each entry records a decision, why it was taken, and what was rejected. Spec references are to
[spec.md](./spec.md); constitution principles are numbered as in `.specify/memory/constitution.md`.

---

## R-001: Where the work happens and what the frontend gets

- **Decision**: All code lands in `hl-budget-backend` on a new branch `002-spending-tracking`,
  cut from `origin/main`, in a git worktree at `hl-budget-spec/hl-budget-backend/`. That is the
  same arrangement as 001 R-001. The worktree there is currently on
  `fix/api-docs-off-in-cluster`, so that fix is merged (or deliberately parked) first. The only
  change in `hl-budget-frontend` is mechanical: copy the regenerated `contracts/api.yaml` and run
  `pnpm generate:api`, so `app/lib/api-schema.d.ts` matches. No page, component or data function
  changes. One small chart change in `hl-helm` adds two environment variables (R-006).
- **Rationale**: The spec is backend-only. The constitution (Development Workflow) says "the
  frontend regenerates its API types in the same feature". Regenerating keeps the frontend's
  `generate:api` diff check green and lets the next feature start from current types.
- **Alternatives**: Leave the frontend untouched. Rejected: its CI diff check would only fail
  later, inside the UI feature, for a reason unrelated to that feature.

## R-002: Who is calling: from token to `users.id`

- **Decision**: A new `CurrentUserService.resolve(identity): UUID` is called first by every
  budgeting service method. It runs one `SELECT id, categories_seeded_at FROM users WHERE
  auth0_subject = :sub`. If there is no row (the caller never hit `PUT /api/v1/users/me`), it runs
  `INSERT … ON CONFLICT (auth0_subject) DO NOTHING`, then the select again. It does not refresh
  name, email or avatar; that stays the job of `PUT /users/me`. Every repository method takes
  the resolved `userId` and every query filters on it.
- **Rationale**: Principle III: identity comes only from the verified token, and all data is
  scoped to the user. Creating the row if missing means the budgeting endpoints do not depend on
  the frontend having called `/users/me` first. The 001 rule "exactly one row per subject" still
  holds because the unique constraint decides.
- **Alternatives**: Answer 404/409 when the user row is missing. Rejected: the client would have
  to know to call `/users/me` first, which is an ordering coupling with no benefit. Putting the
  Auth0 `sub` on every table was also rejected: 001 data-model says future tables reference
  `users(id)`.

## R-003: Default categories, exactly once (FR-010, FR-011, SC-002)

- **Decision**:
  - The 17 names are a constant list in `model/DefaultCategories.kt`, the product's list.
  - `V2` adds `users.categories_seeded_at timestamptz NULL`.
  - Inside `CurrentUserService.resolve`, when that column is null, one transaction runs
    `UPDATE users SET categories_seeded_at = now() WHERE id = :id AND categories_seeded_at IS NULL
    RETURNING id`. Only when a row comes back does it insert the 17 categories with
    `is_default = true`.
  - Under concurrency the second transaction blocks on the row lock, then matches nothing, so it
    inserts nothing.
  - Once seeded, the fast path is the single `SELECT` from R-002.
- **Rationale**: This covers new users and users from before this feature the same way ("first
  use", spec Assumptions). It is race-free without retries, and leaves 001's `PUT /users/me`
  unchanged (FR-041). The flag also keeps working if the default list grows later: a new default
  would come with its own backfill migration, not this path.
- **Alternatives**:
  - Seed inside the 001 upsert, plus a migration backfill for existing users. Rejected: it
    changes a shipped endpoint's behaviour, and a user created between backfill and deploy would
    be missed.
  - Run `INSERT … ON CONFLICT DO NOTHING` on every request. Rejected: 17 conflicting inserts per
    call.
  - A shared global category table. Rejected: the spec says each user has their own categories,
    and every spending FK would then need two kinds of target.

## R-004: Names: trim, case-insensitive uniqueness, 50 characters (FR-013, FR-022)

- **Decision**:
  - The service trims names. The DTO enforces `@NotBlank` and `@Size(max = 50)`.
  - The database adds `CHECK (length(name) BETWEEN 1 AND 50 AND name = btrim(name))` and a unique
    index on `(user_id, lower(name))` for each of `accounts` and `categories`.
  - The service pre-checks with `existsByUserIdAndNameIgnoreCase…` for a good message. It also
    catches `DataIntegrityViolationException` on save and maps both to `DuplicateException`
    (409, `details.reason = "duplicate-name"`, `details.name`), following the skill's "races the
    pre-check cannot win" pattern.
  - The rename pre-check excludes the item itself, so changing only the letter case is allowed.
- **Rationale**: The constraint gives correctness and the pre-check gives the error message. A
  409 is the constitution's status for a uniqueness conflict.
- **Alternatives**: A `citext` column. Rejected: it needs an extension, and `lower()` with an
  expression index is enough at this scale.

## R-005: Refusing deletes that would orphan spendings (FR-025, Clarifications)

- **Decision**:
  - `spendings.account_id` and `spendings.category_id` are foreign keys with `ON DELETE
    NO ACTION` (refuses like `RESTRICT`, but checked at statement end so a future user delete can
    cascade; data-model.md).
  - The service counts the item's spendings first. If the count is above 0 it throws a new
    `ConflictException` in `error/` (409, `details.reason = "in-use"`,
    `details.spendingCount = n`). `DuplicateException` stays reserved for uniqueness clashes, as
    the skill defines it.
  - If a spending is created between the count and the delete, the FK violation is caught and
    mapped to the same 409, with the count read again.
  - A default category is refused before the in-use check: `UnprocessableEntityException` (422,
    `reason = "default-category"`).
- **Rationale**: A 409 means "conflicts with current state", which is what being in use is.
  The database guarantees a spending never loses its account or category, whatever the
  application does.
- **Alternatives**: A 422 for in-use. Rejected: the state can change (move the spendings), so it
  is a conflict and not a fixed rule. The default-category refusal is fixed for all time, so it
  is a 422.

## R-006: Money, the single currency, and "today" (FR-032, FR-030, Clarifications)

- **Decision**:
  - **Storage**: `amount numeric(10,2) NOT NULL CHECK (amount > 0)`, which gives a maximum of
    99,999,999.99. The value is a `BigDecimal` from the entity to the DTO, never a `Double`.
  - **Payload**: money is an object, `{ "amount": "84.37", "currency": "CAD" }`, in both requests
    and responses (Principle VII). In the request DTO `amount.amount` is a `String`, validated by
    `@Pattern("^\d{1,8}(\.\d{1,2})?$")` and then parsed. A JSON number is rejected by type, so a
    float can never be coerced. `amount.currency` is required and must equal the configured
    currency; otherwise the field error is `#/amount/currency`, code `UnsupportedCurrency`.
  - **Zero**: an amount of zero passes the pattern and is refused as a field error, code
    `Positive`.
  - **Configuration**: typed, validated properties `hlbudget.money.currency` (ISO 4217, checked
    with `java.util.Currency` at startup) and `hlbudget.time-zone` (`ZoneId`). They come from
    `HLBUDGET_MONEY_CURRENCY` and `HLBUDGET_TIMEZONE`. Both are required with no default, so a
    missing value fails startup, as with `HLBUDGET_AUTH_*`.
    - Values: `application-local.yaml` and `application-test.yaml` set `CAD` and
      `America/Toronto`. The hl-helm chart sets the same values (the Auth0 tenant is in the
      `ca` region). The operator confirms them before merging.
  - **Today**: a `Clock` bean (UTC system clock; tests inject a fixed one). "Today" is
    `LocalDate.now(clock.withZone(timeZone))`, and a date after it is a field error
    `#/date`, code `NotFuture`.
- **Rationale**: Exact decimals satisfy the constitution's money rule and SC-005. Taking the
  currency in the request now means multi-currency can be added later without a breaking change
  (an optional field becoming required would be breaking). Required config makes the
  "set at deploy time" clarification explicit, not hidden behind a default.
- **Alternatives**:
  - Integer minor units (`amountCents`). Rejected: the constitution's payload rule asks for a
    decimal string. It would be just as exact, but less readable in the contract.
  - A request without a currency. Rejected for the forward-compatibility reason above.
  - Defaults in `application.yaml`. Rejected: a wrong currency is silently baked into every row.

## R-007: Idempotent spending create (FR-036, SC-007)

- **Decision**:
  - An optional `Idempotency-Key` header on `POST /api/v1/spendings`: 1–255 visible ASCII
    characters. A bad value is a 422 field error on the header (R-009).
  - `spendings` stores `idempotency_key varchar(255) NULL` and `idempotency_fingerprint
    char(64) NULL`, which is the SHA-256 of the normalised command (date, trimmed merchant,
    amount, currency, category id, account id). There is a partial unique index on
    `(user_id, idempotency_key) WHERE idempotency_key IS NOT NULL`.
  - On create with a key:
    1. Look up `(userId, key)`. If found with the same fingerprint, return it with **200** and
       `Location`, and add the header `Idempotent-Replayed: true`. If found with a different
       fingerprint, refuse with 422, `reason = "idempotency-key-reused"`.
    2. If not found, insert and return 201. If the insert loses a race on the unique index,
       read the winner and go back to step 1.
  - The key lives exactly as long as the spending: once the spending is deleted, the key can be
    used again.
- **Rationale**: This is the constitution's SHOULD for money-moving creates. Keeping the key on
  the row means no second table and no expiry job (Principle VI). A 200 on replay tells a client
  that nothing new was created, and the body is identical.
- **Alternatives**:
  - A separate `idempotency_keys` table with a TTL. Rejected: it needs a cleanup job for no
    benefit at this scale.
  - Replaying with 201. Rejected: it hides from the client that no new resource was created.

## R-008: Update and delete semantics

- **Decision**:
  - **Update**: `PUT` replaces the whole item. Accounts and categories take `{ name }`.
    Spendings take the same five fields as create. No `PATCH`.
  - **Concurrency**: every table has `version bigint` with `@Version`, so a race inside two
    transactions becomes the existing 409 `concurrent-modification` handler. There is no
    `If-Match`/`ETag`: last write wins between requests, which meets the edge case "neither is
    half-applied".
  - **Delete**: returns 204. A repeat delete is a 404.
  - **Responses**: `PUT` returns 200 with the updated item.
- **Rationale**: Each item has at most five fields, so a full replacement is simple to validate
  and to generate types for. `PUT` is idempotent, as Principle VII requires.
- **Alternatives**:
  - `PATCH` with merge-patch. Rejected: it adds "absent vs null" handling for no user need.
  - `If-Match`. Rejected for a single-user app. It can be added later as an optional header
    without breaking the contract.

## R-009: Status codes: field validation becomes 422

- **Decision**: In `GlobalExceptionHandler`, every response that carries a per-field `errors`
  array becomes **422** `unprocessable-entity`. Today they are 400 `bad-request`.
  - **The 422 cases** (in the body, a query parameter or a header): a constraint failure, a
    missing required field (`Required`), a wrong JSON type for a field (`TypeMismatch`), and an
    unparsable query parameter or header.
  - **The 400 cases**: a request that cannot be read at all. That is a body that is not JSON,
    or a path identifier that is not a UUID (`/spendings/abc`). Neither has a field to point at.
  - **Service-raised field errors** (zero amount, unsupported currency, future date, an unknown
    category or account reference, a bad filter range or sort) use a new
    `ValidationException(violations: List<FieldViolation>)` in `error/`. It extends
    `UnprocessableEntityException` and renders the same `errors` array.
  - The result: a client sees one status for "fix these fields", whichever layer found them.
- **Rationale**: Principle VII lists "`400` malformed request … `422` validation or
  business-rule failure". 001 shipped the 400 mapping, but no published endpoint takes a body or
  documents a validation response (only `PUT /users/me`, which has neither), so the change is
  not breaking. Doing it now, before the first body-taking endpoints, avoids a v2 later.
- **Alternatives**: Keep 400 for consistency with the skill's default. Rejected: the
  constitution wins over the skill (Governance).

## R-010: Listing, filtering, sorting, paging (FR-037–FR-039, SC-004)

- **Decision**:
  - **Endpoint**: `GET /api/v1/spendings` with these parameters:
    - `from` and `to` (`YYYY-MM-DD`, inclusive, each optional)
    - `categoryId` and `accountId` (repeatable UUIDs, OR within a parameter, AND across
      parameters)
    - `page` (0-based, default 0)
    - `size` (default 20, 1–100)
    - `sort`, one of `date,desc` (the default), `date,asc`, `amount,desc`, `amount,asc`
  - **Rejections**: `size` over 100, `from` after `to`, or an unknown `sort` is a 422 field error
    naming the parameter. Spring's silent clamping (`max-page-size`) is not used. Unknown
    category or account ids in a filter just match nothing.
  - **Tie-break**: always `created_at` in the same direction, then `id`. Paging is then stable
    (US4 scenario 4).
  - **Query**: a Spring Data `Specification` on `SpendingEntity`, which always includes
    `user_id = :userId`. The page holds only scalar columns and FK ids. Category and account
    names are joined in the service from the user's own account and category lists. Those lists
    are small (spec Assumptions), which gives 3 queries per page and no N+1.
  - **Response**: `{ "items": [...], "page": 0, "size": 20, "totalItems": 134, "totalPages": 7 }`,
    following the skill's own `PageResponse`, with the items field named per Principle VII.
  - **Indexes**: `(user_id, spending_date DESC, created_at DESC, id)`, `(user_id, category_id)`
    and `(user_id, account_id)`. The last two also serve R-011 and R-005.
- **Rationale**: 10,000 rows per user, indexed by `user_id` first, keeps each page and its count
  in single-digit milliseconds on Postgres, well under SC-004's 1 s. Explicit rejection is what
  the spec's edge cases ask for.
- **Alternatives**:
  - Cursor paging. Rejected: the spec asks for a total of pages, and offset paging is fine at
    this size.
  - A fetch join for names. Rejected: it breaks the count query and returns more columns than
    needed.

## R-011: Spending counts on accounts and categories (FR-026)

- **Decision**: The list and fetch-one queries for accounts and categories use one JPQL query
  each, with a `LEFT JOIN` to an aggregated count of spendings by FK for the same user. The
  results are mapped to `model.Account(id, name, spendingCount)` and
  `model.Category(id, name, isDefault, spendingCount)`. The count is never stored, so it cannot
  drift. Lists are sorted by `lower(name)`, then `id`.
- **Rationale**: Always current, with no counter to keep in step on every spending write, and
  cheap thanks to the R-010 indexes.
- **Alternatives**: A `spending_count` column maintained by the service or a trigger. Rejected:
  it risks drift on concurrent writes, and it is a cache the constitution's Principle VI says to
  avoid.

## R-012: Response shape of a spending (FR-034, FR-035)

- **Decision**:
  ```json
  {
    "id": "uuid",
    "date": "2026-10-20",
    "merchant": "Loblaws",
    "amount": { "amount": "84.37", "currency": "CAD" },
    "category": { "id": "uuid", "name": "Groceries" },
    "account": { "id": "uuid", "name": "Amex" },
    "createdAt": "2026-10-27T14:03:11Z",
    "updatedAt": "2026-10-27T14:03:11Z"
  }
  ```
  The request uses `categoryId` and `accountId`. An unknown id, or another user's id, in the
  request is a 422 field error on that pointer (`#/categoryId`, code `NotFound`), not a 404. The
  404 status is kept for the addressed resource in the path.
- **Rationale**: A 404 means the URL's resource is missing. A bad reference inside a valid
  request is a validation failure on one field, and reporting it by pointer is what FR-005 asks
  for. The response looks the same for "does not exist" and "belongs to someone else", so
  nothing leaks (FR-002).
- **Alternatives**: Respond 404 for a bad reference. Rejected: the client cannot tell which
  reference was wrong, and it reads as "the spending endpoint is missing".

## R-013: Tests

- **Decision**:
  - **Unit tests (MockK, no Spring)**: `CurrentUserService` (seed or skip), the `Account`,
    `Category` and `Spending` services (each refusal and its `reason`), money parsing and the
    currency check, the future-date check with a fixed `Clock`, the idempotency fingerprint,
    sort parsing.
  - **Integration tests (Testcontainers Postgres 17, `RestTestClient`, WireMock JWKS from 001)**:
    one `*ControllerIT` per resource covering every status in the contract. Plus:
    - `DefaultCategoriesIT`: a new user, a pre-existing user, and 10 parallel first calls give
      exactly 17.
    - `OwnershipIT`: user B against every endpoint of user A's items gives 404 or a 422
      `NotFound` pointer, and A's data is unchanged (SC-003).
    - `IdempotencyIT`: 10 parallel identical creates give 1 row; same key with a different body
      gives 422 (SC-007).
    - `SpendingListIT`: 30 rows; filters, sorts, paging with each row seen exactly once, and the
      rejections.
    - `DeleteInUseIT`: account and category deletes give 409 with the count; moving the
      spending then lets the delete succeed.
    - `SpendingListPerformanceIT`: 10,000 rows inserted in one batch, a filtered page in < 1 s
      (SC-004). It runs inside `./gradlew check`, with no tag or flag, because the batch insert
      takes seconds.
  - `SchemaValidationIT` and `OpenApiExportIT` from 001 cover the new entities and assert every
    new operation documents its error responses (SC-006).
- **Rationale**: Principle IV, with coverage per package at 75%. The spec's cross-user and
  concurrency criteria can only be proven against real Postgres.
