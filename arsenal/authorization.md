# Lens: Authorization & IDOR

The territory Brakeman cannot reach: whether the code *checks who is allowed to do what*. Brakeman parses for injection and unsafe APIs; it cannot know that `Order.find(params[:id])` should have been `current_user.orders.find(params[:id])`. That judgment — reading the code with the app's domain in mind — is this lens.

Apply it to every controller that touches user-owned or money-related data, plus the routes file. This lens *covers* authorization; it does not *guarantee* it — say so in the report.

## Methodology

1. **Map the attack surface first.** Read `config/routes.rb` and list every reachable action. Blanket `resources :orders` exposes seven actions — flag routes for actions that don't exist in the controller or were never meant to be public. Generated apps almost never restrict with `only:`. A legacy catch-all route (`match ':controller(/:action(/:id))'`) exposes every public method as an action — flag it outright. For authorization errors on guessable resources, returning 403 (vs 404) confirms the record exists — prefer 404 to avoid enumeration.
2. **Identify the authorization layer.** Pundit, CanCanCan, action_policy, or nothing? If a library is present, the question becomes "where is it *skipped*?" (`authorize` missing in an action, no `verify_authorized` hook, `skip_authorization` sprinkled around). If nothing is present, every controller action is hand-audited.
3. **For each sensitive action ask three questions:** Who can reach this? What records can it load? Is the record scoped to the requester?

## What to Flag Aggressively

**IDOR — the number one finding in unreviewed Rails apps:**

- `Model.find(params[:id])` on user-owned data, instead of scoping through ownership: `current_user.orders.find(...)`. The canonical critical case: a payments/orders/invoices controller where changing the URL id reads someone else's record.
- Scoping on `show` but not on `update`/`destroy` (or vice versa) — auditors check the read path and forget the write path.
- IDOR in nested or non-obvious params: `params[:order_id]` inside a different controller, ids accepted in POST bodies, ids in API serializer links.
- Sequential integer ids make IDOR trivially *discoverable*; UUIDs reduce discoverability but do **not** fix the vulnerability — never accept "we use UUIDs" as the remedy.

```ruby
# Problem — IDOR: any logged-in user reads any invoice by changing the id in the URL
@invoice = Invoice.find(params[:id])              # loaded by id alone, not scoped to the requester

# Fix — scope through ownership; an unowned id now raises RecordNotFound (404), not a leak
@invoice = current_user.invoices.find(params[:id])
```

**Missing or bypassed authorization:**

- Actions with authentication (`authenticate_user!`) but no *authorization* — logged-in is not allowed-to.
- Admin functionality guarded only by obscurity: an `/admin` namespace with no role check, or a `before_action` checking `params` instead of the session user.
- **Records leaked through form helpers**: a `collection_select`/dropdown populated with `Category.all` instead of `policy_scope(Category)` (or `current_user.categories`) exposes every record's existence right in the rendered page — an authorization leak no static scanner sees. Check select boxes and association pickers on user-facing forms.
- Authorization enforced in the **view** (hiding the button) but not in the controller — the request still works via curl.
- Role/permission fields reachable through mass assignment: `permit(:role)`, `permit!`, or `update(params[:user])` on a model with `admin`/`role` columns. Brakeman flags some of this; confirm the semantic cases it can't.
- **`accepts_nested_attributes_for` without ownership checks** — nested params let a user update or delete child records by id (`comments_attributes: [{id: 999, ...}]`) without the controller ever verifying the child belongs to the parent they own. A classic IDOR-through-nesting that static analysis misses: confirm the parent is loaded through `current_user` and that permitted nested ids are re-scoped.
- **Strong-parameters bypassed by a raw Hash** — `User.new(JSON.parse(params[:user]))` (or any `.new`/`.update` fed a plain Hash built outside `ActionController::Parameters`) sidesteps permit entirely, re-opening mass assignment. Common in hand-written JSON API actions and AI-generated endpoints. Trace every `.new`/`.update`/`.assign_attributes` whose argument isn't a `permit`-ed params object. On Rails 8, the idiomatic, type-safe remedy is `params.expect(user: [:name, :email])` — it raises on the array/hash param-type tricks that slip past `require().permit()`.
**Structural signals that raise suspicion:**

- `skip_before_action :authenticate_user!` (or pundit skips) with a long `only:`/`except:` list — those lists drift out of date.
- Authorization logic duplicated inline across actions instead of centralized in policies — inconsistency is where the holes live.
- Webhooks and internal/API endpoints exempted from CSRF and auth "because it's internal", with no signature verification in its place.

## Multi-tenant isolation (cross-tenant IDOR)

In a multi-tenant app (SaaS with `account`/`organization`/`workspace` scoping) the worst finding isn't one user reading another user's row — it's one *tenant* reading another tenant's entire dataset. The signal is always the same: isolation that rests on developer discipline per query rather than a structural guarantee. One forgotten `where(account_id:)` is the whole breach.

- **Tenant resolved from a request-controlled value instead of the session.** Tenant taken from `params`, a request header, or a `Host`/subdomain the client sets — `Current.tenant = Account.find(params[:account_id])` — lets an attacker simply name someone else's tenant. The tenant must come from the authenticated session (`current_user.account`), never from input.
- **Per-query scoping instead of a default guarantee.** Hand-written `where(account: current_account)` on every query is one missed call away from a leak — and the miss is invisible in review. Look for the *absence* of a structural guarantee (`acts_as_tenant`, `ActsAsTenant.current_tenant`, a `default_scope`, or a multitenant gem) on tenant-owned models. A bare `Project.find(params[:id])` in a tenant app is a cross-tenant IDOR.
- **`unscoped` / `default_scope` override on a reachable path.** `unscoped`, `Model.default_scoped(false)`, or a `with_tenant`/admin scope used inside a user-facing action defeats the isolation that exists elsewhere — grep for it and confirm none sit on a request path.
- **Child/join tables without their own tenant column.** Scoping the parent (`current_account.projects`) but loading a child directly (`Task.find(params[:id])`, `comments_attributes` by id) when the child carries no `account_id` of its own — the child is reachable across tenants even though the parent is locked down.
- **Background jobs and callbacks that lose tenant context.** `Current.tenant`/`ActsAsTenant.current_tenant` is request-local; an ActiveJob/Sidekiq worker that doesn't re-establish it runs **unscoped** and can read or mutate every tenant's data. Confirm the tenant is passed as a job argument and re-set in `perform`, not assumed.
- **Global lookups that skip the tenant entirely** — `find_by(email:)` for login, token lookups, or `find_by!(slug:)` that match across tenants when they should be scoped within one.

```ruby
# Problem — tenant taken from a request param, and isolation rests on per-query discipline
ActsAsTenant.current_tenant = Account.find(params[:account_id])   # attacker names any account
@projects = Project.where(account_id: params[:account_id])        # one forgotten scope = full leak

# Fix — tenant from the authenticated session, isolation enforced structurally
set_current_tenant(current_user.account)   # acts_as_tenant scopes every query automatically
@projects = Project.all                     # implicitly scoped to the current tenant
```

State the blast radius explicitly in the report: cross-tenant access is "any customer can read/modify every other customer's data", which sits at the very top alongside IDOR on money paths.

## Severity

Confirmed IDOR or missing authorization on money, PII, or account-takeover paths is **the top of the entire report** — above every CVE and every quality finding. For each confirmed finding, state the exploit in one sentence ("any logged-in user can read any other user's invoices by changing the id"). If you cannot demonstrate the unscoped path, it goes to "theoretical", never to "confirmed".

Untested authorization rules are a finding even when the code looks correct — they're one refactor away from silent failure (this overlaps with the quality lens's test-coverage rule; report it on the security side).

## Preferred Remedies

- Scope queries through ownership: `current_user.orders.find(params[:id])` — the fix is usually one line.
- Adopt one authorization library and enforce it globally (`verify_authorized` / `check_authorization`) so a *missing* check fails loudly instead of silently allowing.
- Restrict routes to the actions that exist: `resources :orders, only: [:index, :show]`.
- Replace per-query tenant discipline with a structural guarantee (default tenant scoping).
- Add a request spec per rule: "user A cannot see user B's record" — the cheapest regression insurance in the codebase.
