# Lens: ActiveRecord Correctness

The model reads fine, but the *ActiveRecord usage* is wrong or unsafe. This is the most distinctly-Rails lens — generic agents get the syntax right and the semantics wrong: a side effect in the callback that fires before the row commits, a "first" record that's whatever the database happened to return, an association that quietly orphans rows. rubycritic scores the file's shape; it never tells you the query is non-deterministic or the callback races the transaction.

Apply it to `app/models/**/*.rb` and the query-heavy code in the hotspot files. This pairs with `arsenal/code-quality.md` — that lens owns *structure* (fat models, N+1 hotspots, `default_scope`, callback-driven *workflows*); this one owns *correctness and integrity* of the ActiveRecord calls themselves. Distilled from the Rails guides and the idiomatic patterns in the `activerecord-patterns` rails-skill.

## Side effects in `after_save` instead of `after_commit`

A job, mailer, or external call in `after_save`/`after_create` runs *inside* the transaction. A background worker can pick the job up and `SELECT` the row before it commits (`RecordNotFound`), and if the transaction later rolls back, the email/charge/webhook has already fired and can't be un-sent. Side effects belong in `after_commit`, scoped to the change that should trigger them.

```ruby
# Problem — fires inside the transaction, and on every save (not just the status change you meant)
class Post < ApplicationRecord
  after_save :enqueue_publish
  def enqueue_publish = PublishPostJob.perform_later(id)
end

# Fix — run side effects after the commit is guaranteed, scoped to the actual change
class Post < ApplicationRecord
  after_commit :enqueue_publish, on: :update, if: :saved_change_to_status?
  def enqueue_publish = PublishPostJob.perform_later(id)
end
```

## `where(...).first` without an explicit order

"First" of an unordered relation is whatever the database returned this time — Postgres can hand back a different row on a different day. It hides as a stable bug in development and surfaces in production. When you mean "the latest/earliest", order explicitly; when you mean "one record or nil", use `find_by`.

```ruby
# Problem — non-deterministic: no ORDER BY, so "first" is undefined
post = Post.where(author_id: current_user.id).first
user = User.where(email: params[:email]).first   # also: returns nil silently, then NoMethodError later

# Fix — order when you mean latest/earliest; find_by for the "one or nil" semantic
post = Post.where(author_id: current_user.id).order(created_at: :desc).first
user = User.find_by(email: params[:email])        # find_by! to raise, find(id) for a PK lookup
```

## `has_many` without `dependent:`

The default when a parent is destroyed is to do *nothing* — the children are left pointing at a parent that no longer exists. Orphan rows accumulate, and on user/account deletion that can be a privacy problem (data that should have been erased lingers). Always state the intent.

```ruby
# Problem — destroying a Post silently orphans its comments
class Post < ApplicationRecord
  has_many :comments
end

# Fix — declare what happens to children
class Post < ApplicationRecord
  has_many :comments, dependent: :destroy          # :delete_all (no callbacks), :nullify, :restrict_with_error
end
```

## Polymorphic association without integrity

`belongs_to :commentable, polymorphic: true` cannot carry a foreign-key constraint — the database can't prove `commentable_id` points at a real row of `commentable_type`, so orphans pile up undetected. When there are only a few parent types and integrity matters, prefer separate foreign keys with a real constraint.

```ruby
# Problem — no FK constraint is possible; nothing stops a dangling commentable_id/type pair
class Comment < ApplicationRecord
  belongs_to :commentable, polymorphic: true
end

# Fix (2–3 parent types) — real FKs + an "exactly one parent" validation (and a DB check constraint)
class Comment < ApplicationRecord
  belongs_to :post,  optional: true
  belongs_to :photo, optional: true
  validate :exactly_one_parent
end
# Many divergent types: delegated_type gives narrow per-type tables — but it does NOT restore FK integrity.
```

## `count > 0` / `present?` for a boolean check

To answer "does any row exist", `exists?` issues the minimal query (`SELECT 1 … LIMIT 1`). `count > 0` runs a full `COUNT(*)`; `present?` loads the whole relation into memory first. Fine on a tiny table, a real cost on a large one.

```ruby
# Problem — does more work than a yes/no needs
return if Order.where(user: user).count > 0     # full SELECT COUNT(*)
return if Order.where(user: user).present?      # loads every matching row into memory

# Fix — a terminating existence check
return if Order.exists?(user: user)
```

## `all.each` / `map(&:attr)` over a large table

`Model.all.each` instantiates every row at once — on a big table that's an out-of-memory crash. `map(&:name)` allocates N model objects just to read one column. Page by primary key with `find_each`, and `pluck` when you only need values.

```ruby
# Problem — loads the entire table (and builds throwaway objects)
User.all.each { |u| Mailer.digest(u).deliver_later }   # 500k rows → 500k objects → OOM
names = User.where(active: true).map(&:name)            # N AR objects to read one column

# Fix — batch by primary key; pluck for raw values
User.find_each(batch_size: 1000) { |u| Mailer.digest(u).deliver_later }
names = User.where(active: true).pluck(:name)
```

## Severity and remedies

Rank by what the misuse actually costs:

- **`after_commit` race and `where.first` non-determinism are correctness bugs**, not style — rank them above cosmetic quality when you can show the failure (a job that `RecordNotFound`s under load; a "latest" that returns a random row, or an auth/ownership lookup that picks an arbitrary match). State the failing scenario in one line.
- **Missing `dependent:` and integrity-free polymorphism** are data-integrity findings — medium, higher when the orphaned rows are PII or money (lingering data that an account deletion should have erased is also a privacy finding — cross-check `arsenal/logging.md`/compliance).
- **`exists?` and `find_each`** are performance findings — low by default, but `find_each` on a known-large table is a real OOM risk: rank it up when the table is big.

Remedies are the idioms themselves (no gem to install): `after_commit` with an `on:`/`if:` guard; explicit `order` or `find_by`; an explicit `dependent:`; separate FKs or `delegated_type`; `exists?`; `find_each`/`pluck`. For soft delete, use `discard` rather than a `default_scope` (pairs with the `default_scope` finding in `arsenal/code-quality.md`). Apply both lenses to model hotspots: this one for correctness, code-quality for structure.
