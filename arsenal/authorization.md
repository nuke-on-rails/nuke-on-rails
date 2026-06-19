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

**Missing or bypassed authorization:**

- Actions with authentication (`authenticate_user!`) but no *authorization* — logged-in is not allowed-to.
- Admin functionality guarded only by obscurity: an `/admin` namespace with no role check, or a `before_action` checking `params` instead of the session user.
- **Records leaked through form helpers**: a `collection_select`/dropdown populated with `Category.all` instead of `policy_scope(Category)` (or `current_user.categories`) exposes every record's existence right in the rendered page — an authorization leak no static scanner sees. Check select boxes and association pickers on user-facing forms.
- Authorization enforced in the **view** (hiding the button) but not in the controller — the request still works via curl.
- Role/permission fields reachable through mass assignment: `permit(:role)`, `permit!`, or `update(params[:user])` on a model with `admin`/`role` columns. Brakeman flags some of this; confirm the semantic cases it can't.
- **`accepts_nested_attributes_for` without ownership checks** — nested params let a user update or delete child records by id (`comments_attributes: [{id: 999, ...}]`) without the controller ever verifying the child belongs to the parent they own. A classic IDOR-through-nesting that static analysis misses: confirm the parent is loaded through `current_user` and that permitted nested ids are re-scoped.
- **Strong-parameters bypassed by a raw Hash** — `User.new(JSON.parse(params[:user]))` (or any `.new`/`.update` fed a plain Hash built outside `ActionController::Parameters`) sidesteps permit entirely, re-opening mass assignment. Common in hand-written JSON API actions and AI-generated endpoints. Trace every `.new`/`.update`/`.assign_attributes` whose argument isn't a `permit`-ed params object. On Rails 8, the idiomatic, type-safe remedy is `params.expect(user: [:name, :email])` — it raises on the array/hash param-type tricks that slip past `require().permit()`.
- Multi-tenant apps where tenant scoping depends on developer discipline per-query instead of a default scope/`acts_as_tenant`-style guarantee — one forgotten `where(account:)` is a cross-tenant leak.

**Structural signals that raise suspicion:**

- `skip_before_action :authenticate_user!` (or pundit skips) with a long `only:`/`except:` list — those lists drift out of date.
- Authorization logic duplicated inline across actions instead of centralized in policies — inconsistency is where the holes live.
- Webhooks and internal/API endpoints exempted from CSRF and auth "because it's internal", with no signature verification in its place.

## Severity

Confirmed IDOR or missing authorization on money, PII, or account-takeover paths is **the top of the entire report** — above every CVE and every quality finding. For each confirmed finding, state the exploit in one sentence ("any logged-in user can read any other user's invoices by changing the id"). If you cannot demonstrate the unscoped path, it goes to "theoretical", never to "confirmed".

Untested authorization rules are a finding even when the code looks correct — they're one refactor away from silent failure (this overlaps with the quality lens's test-coverage rule; report it on the security side).

## Preferred Remedies

- Scope queries through ownership: `current_user.orders.find(params[:id])` — the fix is usually one line.
- Adopt one authorization library and enforce it globally (`verify_authorized` / `check_authorization`) so a *missing* check fails loudly instead of silently allowing.
- Restrict routes to the actions that exist: `resources :orders, only: [:index, :show]`.
- Replace per-query tenant discipline with a structural guarantee (default tenant scoping).
- Add a request spec per rule: "user A cannot see user B's record" — the cheapest regression insurance in the codebase.
