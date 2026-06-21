# Weapon: Migration Safety (zero-downtime)

The availability weapon no engine owns. rubycritic scores `app/`, Brakeman hunts exploits — neither reads `db/migrate/`, yet a single careless migration is a production outage: a statement that takes an `ACCESS EXCLUSIVE` lock on a large table, or a schema change deployed ahead of the code that depends on it, takes the app down during the deploy that was supposed to be routine. This finding is about *downtime*, not a code smell.

Apply it to recent migrations (the ones about to ship) in `db/migrate/`, read against `db/schema.rb` for which tables are large. Whether a lock actually hurts depends on row count and traffic, which static analysis can't see — so flag conditionally: "this locks `orders` for the duration of the rewrite; on a large, write-hot table that's an outage." The de-facto structural guarantee is the **strong_migrations** gem (the RuboCop of migrations); its absence on a team shipping migrations is itself worth a line.

## Operations that lock or rewrite the table

- **`add_column` with `null: false` and no default** (or a default on Postgres < 11) — rewrites every row under a lock. The safe shape is three steps: add nullable → backfill in batches → add the `NOT NULL` constraint (validated separately).
- **`add_index` without `algorithm: :concurrently`** — a plain `add_index` blocks writes for the whole build. Concurrent index creation needs `disable_ddl_transaction!` in the migration; flag the pair being absent together.
- **`add_index ..., unique: true` without deduping existing rows first** — if the table already holds duplicates, index creation fails and the migration aborts mid-deploy (often only in the environment that has the dupes). Dedupe in the same migration, immediately before adding the index. In a multi-tenant app the uniqueness is usually per-tenant, so the index is composite (`[:account_id, :email]`), not global. Pairs with the `validates uniqueness` finding in `arsenal/activerecord.md`.
- **`change_column`** that changes type (or `change_column_null` on a big table without a pre-validated check constraint) — full table rewrite under lock.
- **`add_foreign_key`** in one shot — validating the constraint takes a lock proportional to the table. The safe shape is `add_foreign_key ..., validate: false` then `validate_foreign_key` in a later migration.
- **`add_check_constraint`** without `validate: false` — same lock as above.

```ruby
# Problem — both statements take a lock on orders; on a large, write-hot table that's downtime
class AddIndexAndFlagToOrders < ActiveRecord::Migration[8.0]
  def change
    add_index  :orders, :customer_id                      # plain index build blocks writes for its duration
    add_column :orders, :archived, :boolean, null: false  # NOT NULL + existing rows → rewrite/failure
  end
end

# Fix — build the index concurrently; add the column nullable, then backfill + constrain in steps
class AddIndexAndFlagToOrders < ActiveRecord::Migration[8.0]
  disable_ddl_transaction!                                # required for concurrent index creation
  def change
    add_index  :orders, :customer_id, algorithm: :concurrently
    add_column :orders, :archived, :boolean               # nullable: an instant metadata change
    # then, separately: backfill in batches → change_column_null with a pre-validated constraint
  end
end
```

```ruby
add_index :users, :email, unique: true  # Problem — aborts if the table already has duplicate emails
# Fix — dedupe in the same migration, then add the index (composite per tenant if multi-tenant)
execute "DELETE FROM users a USING users b WHERE a.email = b.email AND a.id > b.id"
add_index :users, :email, unique: true
```

## Backfills inside a schema migration

- **Data backfill in a DDL migration** — `User.update_all(...)`, a `find_each` loop, or `execute "UPDATE ..."` inside the migration that changed the schema. It runs inside the DDL transaction (holding locks longer), loads rows into memory, and can't be retried cleanly. Backfills belong in a separate, batched, idempotent migration or rake task — never mixed with the structural change.
- **Referencing application models** (`User`, not a throwaway inline class) in a migration — the model's current code may not match the schema at the point the migration runs on an old deploy; validations and callbacks fire unexpectedly. Migrations should use raw SQL or a minimal inline `Class.new(ActiveRecord::Base)`.

```ruby
# Problem — backfill inside the schema migration: holds locks, loads every row, fires app callbacks
class AddStatusToOrders < ActiveRecord::Migration[8.0]
  def change
    add_column :orders, :status, :string
    Order.find_each { |o| o.update!(status: "legacy") }   # in the DDL transaction, via the app model
  end
end

# Fix — schema change only; backfill separately, in batches, without the app model
class BackfillOrderStatus < ActiveRecord::Migration[8.0]
  disable_ddl_transaction!
  def up
    Order.unscoped.in_batches(of: 5_000) { |batch| batch.update_all(status: "legacy") }
  end
end
```

## Destructive data operations

A migration runs automatically on every deploy that hasn't seen it — there is no confirmation prompt and no easy undo. A destructive data statement inside one (`TRUNCATE`, `DELETE`, `delete_all`/`destroy_all`, a `drop_table` on a table still holding data, or an `execute` that rewrites rows) is irreversible data loss waiting for the next deploy. Schema changes belong in migrations; one-off or destructive *data* changes belong in a rake task you run deliberately, with a backup and a chance to abort.

```ruby
# Problem — wipes a table on deploy, automatically, with no confirmation and no clean rollback
class ResetOrders < ActiveRecord::Migration[8.0]
  def up
    execute "TRUNCATE orders"      # gone the moment this deploy ships
    Order.insert_all(legacy_rows)
  end
end

# Fix — keep destructive data work out of migrations: a guarded, re-runnable rake task you run on
# purpose (after a backup), never a statement the deploy fires by itself
```

## Deploy-ordering hazards (expand/contract)

A rolling deploy runs the migration while old code is still serving requests. Two changes break that window:

- **Removing or renaming a column/table** in the same deploy as the code that stops using it — between migrate and the last old pod draining, old code selects a column that no longer exists and every request errors. `rename_column` is doubly unsafe (old *and* new code break). The safe path is expand/contract across two deploys: stop referencing it, ship, then drop it. `ignored_columns` bridges the gap for removals.
- **Irreversible migration with no `down`** — a `change` block containing `remove_column`/`execute`/raw SQL that can't auto-reverse, with no explicit `down` or `reversible`, means a failed deploy can't roll back cleanly. Flag migrations that aren't safely reversible.

```ruby
# Problem — column dropped in the same deploy as the code that stops using it. Until the last old
# pod drains, old code still SELECTs legacy_status → every request errors mid-deploy.
class RemoveLegacyStatusFromOrders < ActiveRecord::Migration[8.0]
  def change
    remove_column :orders, :legacy_status, :string
  end
end

# Fix — expand/contract: deploy 1 tells the model to ignore the column; drop it in deploy 2
class Order < ApplicationRecord
  self.ignored_columns += ["legacy_status"]   # ship this first; remove_column only after it's fully out
end
```

## Performance (overlaps code-quality)

- **Foreign-key and frequently-filtered columns with no index** — `belongs_to`/`references` whose column has no `add_index`, or a `where`/`order` hot path with no covering index, surfaces as N+1-adjacent slowness. Pairs with `arsenal/code-quality.md`.

```ruby
# Problem — a foreign-key column with no index: every lookup/join on it is a full table scan
add_column :comments, :post_id, :bigint   # added by hand, no index

# Fix — index the foreign key (concurrently on a large table)
add_index :comments, :post_id, algorithm: :concurrently
```

## Severity and remedies

Most findings here are **operational/availability tier** — above cosmetic quality, below demonstrated security exploits. The exception: a lock-taking or column-dropping migration about to ship against a large, traffic-heavy table is a *scheduled outage* — say so plainly and rank it accordingly, because the cost is total downtime, not slower code. State the at-risk table by name and the safe rewrite in one line (split into steps / `algorithm: :concurrently` / `validate: false` then validate / expand-contract). Recommend the **strong_migrations** gem as the structural guarantee so the next unsafe migration fails in CI, not in production.
