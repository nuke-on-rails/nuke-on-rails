# Weapon: Authorization & IDOR

The territory Brakeman cannot reach: whether the code *checks who is allowed to do what*. Brakeman parses for injection and unsafe APIs; it cannot know that `Order.find(params[:id])` should have been `current_user.orders.find(params[:id])`. That judgment — reading the code with the app's domain in mind — is this weapon.

Apply it to every controller that touches user-owned or money-related data, plus the routes file. This weapon *covers* authorization; it does not *guarantee* it — say so in the report.

## Methodology

1. **Map the attack surface first.** Read `config/routes.rb` and list every reachable action. Blanket `resources :orders` exposes seven actions — flag routes for actions that don't exist in the controller or were never meant to be public. Generated apps almost never restrict with `only:`. A legacy catch-all route (`match ':controller(/:action(/:id))'`) exposes every public method as an action — flag it outright. For authorization errors on guessable resources, returning 403 (vs 404) confirms the record exists — prefer 404 to avoid enumeration.
2. **Identify the authorization layer.** Pundit, CanCanCan, action_policy, or nothing? If a library is present, the question becomes "where is it *skipped*?" (`authorize` missing in an action, no `verify_authorized` hook, `skip_authorization` sprinkled around). If nothing is present, every controller action is hand-audited.
3. **For each sensitive action ask three questions:** Who can reach this? What records can it load? Is the record scoped to the requester?

## What to Flag Aggressively

**IDOR — the number one finding in unreviewed Rails apps:**

- `Model.find(params[:id])` on user-owned data, instead of scoping through ownership: `current_user.orders.find(...)`. The canonical critical case: a payments/orders/invoices controller where changing the URL id reads someone else's record.

  ```ruby
  @invoice = Invoice.find(params[:id])               # Problem — any logged-in user reads any invoice by id
  @invoice = current_user.invoices.find(params[:id]) # Fix — scoped to the owner; an unowned id 404s
  ```
- Scoping on `show` but not on `update`/`destroy` (or vice versa) — auditors check the read path and forget the write path.

  ```ruby
  def update = Order.find(params[:id]).update(order_params)               # Problem — write path unscoped
  def update = current_user.orders.find(params[:id]).update(order_params) # Fix — scope every action alike
  ```
- IDOR in nested or non-obvious params: `params[:order_id]` inside a different controller, ids accepted in POST bodies, ids in API serializer links.

  ```ruby
  LineItem.find(params[:line_item_id])  # Problem — id in a nested/POST param, unscoped
  current_user.orders.find(params[:order_id]).line_items.find(params[:line_item_id])  # Fix — via the owner
  ```
- Sequential integer ids make IDOR trivially *discoverable*; UUIDs reduce discoverability but do **not** fix the vulnerability — never accept "we use UUIDs" as the remedy.

**Missing or bypassed authorization:**

- Actions with authentication (`authenticate_user!`) but no *authorization* — logged-in is not allowed-to.

  ```ruby
  authenticate_user!; Post.find(params[:id]).destroy      # Problem — logged in ≠ allowed; anyone deletes any post
  authorize(@post = Post.find(params[:id])); @post.destroy # Fix — Pundit checks PostPolicy#destroy?
  ```
- Admin functionality guarded only by obscurity: an `/admin` namespace with no role check, or a `before_action` checking `params` instead of the session user.

  ```ruby
  namespace(:admin) { resources :users }   # Problem — /admin reachable by anyone, no role check
  authenticate(:user, ->(u) { u.admin? }) { namespace(:admin) { resources :users } }  # Fix — gate on role
  ```
- **Records leaked through form helpers**: a `collection_select`/dropdown populated with `Category.all` instead of `policy_scope(Category)` (or `current_user.categories`) exposes every record's existence right in the rendered page — an authorization leak no static scanner sees. Check select boxes and association pickers on user-facing forms.

  ```ruby
  collection_select :category_id, Category.all            # Problem — exposes every category's existence
  collection_select :category_id, policy_scope(Category)  # Fix — only what the user may see
  ```
- **Cache keys not scoped to the requester**: `Rails.cache.fetch("dashboard") { current_user.dashboard }`, or a fragment `<% cache "sidebar" %>` wrapping per-user content, serves the first user's data to everyone who hits it next — a cross-user leak no scanner sees. Any cache (fragment or low-level) around user-specific data must include the user/account in the key (`[current_user, ...]`).

  ```ruby
  Rails.cache.fetch("dashboard") { current_user.dashboard }                 # Problem — one user's data to everyone
  Rails.cache.fetch([current_user, :dashboard]) { current_user.dashboard }  # Fix — keyed per user
  ```
- **Turbo Stream broadcasts rendered for the wrong audience**: `broadcasts_to` renders the partial once, with no request context (no `current_user`), and sends the *same* HTML to every subscriber of the stream. A partial with viewer-specific content — an owner-only Edit/Delete button, another user's private fields — either breaks or leaks to everyone subscribed. Only `turbo_stream_from` a stream on pages where the viewer may receive every future broadcast to it.

  ```ruby
  broadcasts_to ->(comment) { [comment.post, :comments] }  # Problem — one partial (no current_user) to all subscribers
  # Fix — broadcast only what every subscriber may see; keep per-viewer controls out of the partial
  # (render them client-side, or use a per-recipient stream) — never current_user-dependent markup
  ```
- **ActionCable channels that subscribe without ownership scoping**: a `Channel#subscribed` that does `stream_for Room.find(params[:id])` lets any authenticated user subscribe to any room's stream by id and receive every broadcast to it — IDOR over the websocket (messages, presence) on a resource they don't belong to. Scope the lookup through the requester; authenticate `Connection#connect` with the same identity resolution as HTTP.

  ```ruby
  def subscribed; stream_for Room.find(params[:id]); end               # Problem — any user streams any room by id
  def subscribed; stream_for current_user.rooms.find(params[:id]); end # Fix — scoped to the subscriber's rooms
  ```
- Authorization enforced in the **view** (hiding the button) but not in the controller — the request still works via curl.

  ```ruby
  # Problem — the view hides the button, but `def destroy = @post.destroy` still runs via curl
  # Fix — enforce in the controller too: `authorize @post` before destroying
  ```
- Role/permission fields reachable through mass assignment: `permit(:role)`, `permit!`, or `update(params[:user])` on a model with `admin`/`role` columns. Brakeman flags some of this; confirm the semantic cases it can't.

  ```ruby
  params.require(:user).permit(:email, :role)   # Problem — user can set role: "admin"
  params.require(:user).permit(:email)          # Fix — never permit privilege columns from params
  ```
- **`accepts_nested_attributes_for` without ownership checks** — nested params let a user update or delete child records by id (`comments_attributes: [{id: 999, ...}]`) without the controller ever verifying the child belongs to the parent they own. A classic IDOR-through-nesting that static analysis misses: confirm the parent is loaded through `current_user` and that permitted nested ids are re-scoped.

  ```ruby
  Post.find(params[:id]).update(post_params)               # Problem — nested comment ids unscoped to the owner
  current_user.posts.find(params[:id]).update(post_params) # Fix — load the parent through ownership
  ```
- **Strong-parameters bypassed by a raw Hash** — `User.new(JSON.parse(params[:user]))` (or any `.new`/`.update` fed a plain Hash built outside `ActionController::Parameters`) sidesteps permit entirely, re-opening mass assignment. Common in hand-written JSON API actions and AI-generated endpoints. Trace every `.new`/`.update`/`.assign_attributes` whose argument isn't a `permit`-ed params object. On Rails 8, the idiomatic, type-safe remedy is `params.expect(user: [:name, :email])` — it raises on the array/hash param-type tricks that slip past `require().permit()`.

  ```ruby
  User.new(JSON.parse(params[:user]))    # Problem — bypasses permit; mass assignment reopened
  params.expect(user: [:name, :email])   # Fix (Rails 8) — type-safe allowlist
  ```

**Structural signals that raise suspicion:**

- `skip_before_action :authenticate_user!` (or pundit skips) with a long `only:`/`except:` list — those lists drift out of date.

  ```ruby
  skip_before_action :authenticate_user!, except: %i[show index]  # Problem — list drifts; new actions ship public
  # Fix — default closed: authenticate globally, open only the few public actions explicitly (only:)
  ```
- Authorization logic duplicated inline across actions instead of centralized in policies — inconsistency is where the holes live.

  ```ruby
  redirect_to(root_path) unless @post.author == current_user  # Problem — copy-pasted per action, drifts
  authorize @post                                             # Fix — one PostPolicy#update?, enforced in one place
  ```
- Webhooks and internal/API endpoints exempted from CSRF and auth "because it's internal", with no signature verification in its place.

  ```ruby
  skip_before_action :verify_authenticity_token   # Problem — "internal", no signature check in its place
  # Fix — verify the signature instead (see arsenal/api.md)
  ```

## Multi-tenant isolation (cross-tenant IDOR)

In a multi-tenant app (SaaS with `account`/`organization`/`workspace` scoping) the worst finding isn't one user reading another user's row — it's one *tenant* reading another tenant's entire dataset. The signal is always the same: isolation that rests on developer discipline per query rather than a structural guarantee. One forgotten `where(account_id:)` is the whole breach.

- **Tenant resolved from a request-controlled value instead of the session.** Tenant taken from `params`, a request header, or a `Host`/subdomain the client sets — `Current.tenant = Account.find(params[:account_id])` — lets an attacker simply name someone else's tenant. The tenant must come from the authenticated session (`current_user.account`), never from input.

  ```ruby
  ActsAsTenant.current_tenant = Account.find(params[:account_id])  # Problem — attacker names any account
  set_current_tenant(current_user.account)                         # Fix — tenant from the authenticated session
  ```
- **Per-query scoping instead of a default guarantee.** Hand-written `where(account: current_account)` on every query is one missed call away from a leak — and the miss is invisible in review. Look for the *absence* of a structural guarantee (`acts_as_tenant`, `ActsAsTenant.current_tenant`, a `default_scope`, or a multitenant gem) on tenant-owned models. A bare `Project.find(params[:id])` in a tenant app is a cross-tenant IDOR. And `acts_as_tenant` only *fails closed* when `require_tenant = true` — without it (the default), a query that loses the tenant context silently returns **every** tenant's rows instead of raising, so the gem's mere presence is false confidence.

  ```ruby
  Project.where(account_id: current_account.id).find(params[:id])  # Problem — one forgotten scope = leak
  Project.find(params[:id])  # Fix — acts_as_tenant scopes every query (set require_tenant = true so it fails closed)
  ```
- **`unscoped` / `default_scope` override on a reachable path.** `unscoped`, `Model.default_scoped(false)`, or a `with_tenant`/admin scope used inside a user-facing action defeats the isolation that exists elsewhere — grep for it and confirm none sit on a request path.

  ```ruby
  Project.unscoped.find(params[:id])           # Problem — defeats tenant isolation on a request path
  current_account.projects.find(params[:id])   # Fix — stay within the tenant scope
  ```
- **Child/join tables without their own tenant column.** Scoping the parent (`current_account.projects`) but loading a child directly (`Task.find(params[:id])`, `comments_attributes` by id) when the child carries no `account_id` of its own — the child is reachable across tenants even though the parent is locked down.

  ```ruby
  Task.find(params[:id])  # Problem — child reachable cross-tenant (no account_id of its own)
  current_account.projects.find(params[:project_id]).tasks.find(params[:id])  # Fix — through the scoped parent
  ```
- **Background jobs and callbacks that lose tenant context.** `Current.tenant`/`ActsAsTenant.current_tenant` is request-local; an ActiveJob/Sidekiq worker that doesn't re-establish it runs **unscoped** and can read or mutate every tenant's data. Confirm the tenant is passed as a job argument and re-set in `perform`, not assumed.

  ```ruby
  def perform(id) = Project.find(id).archive!   # Problem — runs unscoped, reaches every tenant's data
  def perform(account_id, id)                   # Fix — pass the tenant and re-establish it
    ActsAsTenant.with_tenant(Account.find(account_id)) { Project.find(id).archive! }
  end
  ```
- **Global lookups that skip the tenant entirely** — `find_by(email:)` for login, token lookups, or `find_by!(slug:)` that match across tenants when they should be scoped within one.

  ```ruby
  User.find_by(email: params[:email])                    # Problem — matches across all tenants
  current_account.users.find_by(email: params[:email])   # Fix — scoped within the tenant
  ```

State the blast radius explicitly in the report: cross-tenant access is "any customer can read/modify every other customer's data", which sits at the very top alongside IDOR on money paths.

## Severity

Confirmed IDOR or missing authorization on money, PII, or account-takeover paths is **the top of the entire report** — above every CVE and every quality finding. For each confirmed finding, state the exploit in one sentence ("any logged-in user can read any other user's invoices by changing the id"). If you cannot demonstrate the unscoped path, it goes to "theoretical", never to "confirmed".

Untested authorization rules are a finding even when the code looks correct — they're one refactor away from silent failure (this overlaps with the quality weapon's test-coverage rule; report it on the security side).

## Preferred Remedies

- Scope queries through ownership: `current_user.orders.find(params[:id])` — the fix is usually one line.
- Adopt one authorization library and enforce it globally (`verify_authorized` / `check_authorization`) so a *missing* check fails loudly instead of silently allowing.
- Restrict routes to the actions that exist: `resources :orders, only: [:index, :show]`.
- Replace per-query tenant discipline with a structural guarantee (default tenant scoping).
- Add a request spec per rule: "user A cannot see user B's record" — the cheapest regression insurance in the codebase.
