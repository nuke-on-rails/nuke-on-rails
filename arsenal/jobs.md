# Weapon: Background Job Safety

Background jobs are infrastructure with three properties that bite: they **retry by design**, they **persist their arguments**, and they **run later** against data that may have moved. The failures are invisible until a retry double-charges a customer, or a job argument turns out to be a password sitting in the jobs table. No engine models this — Brakeman sees a method call, not a side effect that runs twice.

Apply it to `app/jobs/**/*.rb`, Sidekiq/`ActiveJob` classes, and the enqueue sites. Cross-references: `arsenal/activerecord.md` (enqueue from `after_commit`, not `after_save`), `arsenal/api.md` (webhook idempotency, the same problem on the inbound side), and `arsenal/hardening.md` (the job dashboard that exposes these arguments).

## Non-idempotent jobs (a retry repeats the side effect)

A job can run more than once — a network blip after the work but before the ack, a worker crash, a manual replay. If `perform` charges, provisions, emails, or mutates with no guard, the second run does it again. Every job that causes an external or irreversible effect needs an idempotency strategy: a state check, an external idempotency key, or a unique constraint.

```ruby
# Problem — jobs retry by design; if the charge succeeds but the network drops, the retry charges again
class ChargeCustomerJob < ApplicationJob
  def perform(order_id)
    order = Order.find(order_id)
    Stripe::PaymentIntent.create(amount: order.total, currency: "usd",
                                 payment_method: order.payment_method_id, confirm: true)
    order.update!(status: "paid")
  end
end

# Fix — guard on state, and pass an idempotency key so a retry is a no-op on Stripe's side too
def perform(order_id)
  order = Order.find(order_id)
  return if order.paid?
  intent = Stripe::PaymentIntent.create(
    { amount: order.total, currency: "usd", payment_method: order.payment_method_id, confirm: true },
    idempotency_key: "order-#{order.id}-charge"
  )
  order.update!(stripe_payment_intent_id: intent.id, status: "paid")
end
```

## Secrets or PII in job arguments

Job arguments are serialized and **persisted** — in the Solid Queue tables (your own database) or in Redis for Sidekiq — and they show up in the job dashboard (Mission Control / Sidekiq Web, often mounted without auth — see `arsenal/hardening.md`) and in the `enqueue` log line. A raw password, token, card number, or full PII record passed as an argument leaks into all three. Pass an id or a single-use token; let the job load or mint what it needs.

```ruby
# Problem — the raw password is now in the jobs table / Redis, on the dashboard, and in the logs
ResetPasswordJob.perform_later(user.id, raw_password)

# Fix — pass the id; the job generates and emails the reset token itself
ResetPasswordJob.perform_later(user.id)
```

## Stale data: passing a record instead of its id

Serializing a whole ActiveRecord object into a job freezes its state at enqueue time — by the time the worker runs, the row may have changed, and a large object bloats the queue store (or fails to deserialize after a deploy that changed the schema). Pass the id and reload inside `perform`.

```ruby
# Problem — the record is serialized now; the worker acts on stale attributes later
NotifyJob.perform_later(order)

# Fix — pass the id, reload inside perform so the job sees current data
NotifyJob.perform_later(order.id)
def perform(order_id) = Order.find(order_id).notify
```

Exception: a job enqueued from `after_destroy_commit` **can't** reload the record — it's already gone, so `Order.find(id)` raises `RecordNotFound`. Pass the data the job needs as plain arguments, captured before the destroy.

```ruby
after_destroy_commit { CleanupJob.perform_later(id, owner_email) }  # Fix — snapshot what the job needs
```

## Severity and remedies

Rank by what the misuse costs:

- **A non-idempotent job on a money or provisioning path is correctness-and-money** — rank it high when you can show the retry that repeats the effect (Stripe succeeds, the network drops, the retry charges again). State that path in one line.
- **Secrets/PII in job arguments is a disclosure finding** — the queue store and the (frequently unauthenticated) dashboard expose it; rank by data class, and pair it with the exposed-dashboard check in `arsenal/hardening.md`.
- **Object-instead-of-id is correctness/robustness** — lower, but also a deserialization-failure footgun across deploys.

Remedies are the idioms, no gem required: a state guard plus an external idempotency key (or a DB unique constraint / an idempotency-log table) for at-least-once safety; pass ids — never secrets or whole objects — and load inside `perform`; enqueue from `after_commit` (see `arsenal/activerecord.md`) so the worker never races the transaction. Solid Queue's `limits_concurrency` and Sidekiq's `sidekiq-unique-jobs` prevent duplicate *enqueues*, but they are not a substitute for an idempotent `perform`.
