---
name: database-design
description: Use when creating or changing tables, columns, migrations, models or relationships, storing money, dates, statuses, files or user-facing IDs, or changing a large table that is already in production. Covers money types, foreign keys and delete rules, soft deletes with unique constraints, enums, timezones, IDs, JSON columns, price snapshots and zero-downtime migrations, with Laravel examples. Query speed and indexes are the `performance` skill; migration hygiene (reversible, never edited after release) is the `security` skill.
---

> Scope: this skill covers the shape of the data: types, keys, constraints and how a schema changes safely. Indexes and query cost are the `performance` skill. Migration hygiene (versioned, reversible, immutable after release) is the `security` skill. A schema mistake outlives the code that made it, so get it right before the first row is written.

## 1. Money
- Never store money in `float` or `double`. `0.1 + 0.2` is not `0.3`, and the error ends up in invoices.
- Store it as an integer in the smallest unit (`amount` in cents, `unsignedBigInteger`), or as `decimal('amount', 12, 2)`. Pick one per project and use it everywhere.
- Store the currency next to the amount (`char('currency', 3)`, ISO 4217) as soon as more than one currency is possible.
- Do the arithmetic in integers or a money library, not in floats, and round once, at a defined step (per line or per total), the same way every time.

## 2. Snapshot What the Business Agreed To
- An order line stores the `unit_price`, `tax_rate` and product name **at the time of the order**. It never reads the current product price later. Prices change; invoices must not.
- The same goes for anything a customer saw and accepted: shipping fee, discount, exchange rate, address.

## 3. Keys and Relationships
- Every relationship column has a real foreign key: `$table->foreignId('user_id')->constrained()`.
- Choose the delete rule on purpose:
  - `->cascadeOnDelete()` for data that has no meaning without its parent (a cart's items).
  - `->restrictOnDelete()` for records the business must keep (orders, payments, invoices): the parent can't be deleted while they exist.
  - `->nullOnDelete()` when the child survives on its own (a post whose author left).
- Polymorphic relations (`morphTo`) can't have a foreign key. Use them only when the set of parent types is truly open; otherwise use a normal relation per type.
- Many-to-many pivot tables get a unique index on the pair, so the same link can't exist twice.

## 4. IDs
- Internal tables use the default auto-increment `id`.
- An ID that appears in a URL or API and must not be guessable or counted (orders, invoices, files, users) uses a ULID or UUID column, either as the key (`HasUlids`) or as a separate `public_id` with a unique index. Sequential IDs leak volume and invite IDOR probing; the `security` skill still requires the ownership check.

## 5. Constraints Are the Last Line of Defense
- Rules the data must never break live in the database, not only in validation: `unique` for emails, slugs, idempotency keys and "one active subscription per user"; `NOT NULL` for required fields; foreign keys for relationships.
- Validation gives a friendly error; the constraint guarantees it under concurrency.
- **Soft deletes break plain unique indexes.** A soft-deleted user still holds the email, so the same person can't sign up again. Adding `deleted_at` to the unique index does not fix it: MySQL and PostgreSQL treat NULLs as distinct, so two active rows with the same email would both be allowed. Decide up front:
  - PostgreSQL: a partial unique index on `email` where `deleted_at IS NULL`.
  - MySQL: a generated column that holds the email only while the row is active (`NULL` once deleted), with a unique index on that column.
  - Or don't soft-delete that table.

## 6. Nulls and Defaults
- A column is nullable only when "unknown" or "not set" is a real, valid state. Otherwise it is `NOT NULL`, with a default when one makes sense.
- Booleans are never nullable: three states hiding in a boolean is a bug.
- A new `NOT NULL` column on a table with data needs a default or a backfill before the constraint (see section 9).

## 7. Statuses and Enums
- Store statuses as a string column and cast them to a PHP backed enum on the model (`'status' => OrderStatus::class` in `casts`). Avoid the database `ENUM` type: adding a value means altering the table.
- Allowed transitions belong to the model (see State Machines in the `architecture` skill), not to the column.

## 8. Dates and Time
- The app and the database run in UTC (`'timezone' => 'UTC'` in `config/app.php`). Convert to the user's timezone only when displaying.
- Store the user's timezone on the user when the app shows times to people in more than one zone.
- A calendar date with no time (a birthday, a due date) is a `date` column, not a timestamp at midnight.
- On MySQL, `timestamp` columns end in 2038. Use `dateTime` for dates that can be far in the future (expiry dates, long contracts).

## 9. Changing Tables in Production
A migration that is fine on a laptop can lock a production table with millions of rows for minutes.

- **Expand, then contract.** Never rename or drop a column in the same deploy as the code that stops using it:
  1. Add the new column (nullable or with a default) and deploy code that writes to both.
  2. Backfill existing rows in a queued job or command, in chunks (`chunkById`), not inside the migration.
  3. Switch reads to the new column, then add `NOT NULL` or constraints.
  4. Drop the old column in a later deploy.
- Data backfills never run inside a schema migration on a large table: they hold locks and can't be retried safely.
- Before running a migration on a large table, check it against a copy with production-size data and know how long it locks.

## 10. JSON Columns
- Use a JSON column for data that is truly free-form and read as a whole (provider payloads, user preferences, metadata).
- Anything you filter, sort, join or validate on gets its own column. Querying inside JSON is slow and has no constraints.

## 11. Before You Call It Done
- Run `php artisan migrate:fresh --seed` and then `php artisan migrate:rollback` on the new migrations; both must succeed.
- Inspect the result with `php artisan db:table <table>` and confirm every type, nullability, default, foreign key and unique index is what you intended.
- Grep the diff for `float(`, `double(`, `->enum(`, and `foreignId(` without `constrained(`.
- A test that the database rejects what it must: a duplicate unique value, a missing required field, deleting a parent with restricted children.
- For a change to an existing large table, state the deploy steps (expand, backfill, contract) and what each one locks.
- Report what you verified and what you didn't.
