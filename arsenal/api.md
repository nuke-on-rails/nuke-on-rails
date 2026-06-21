# Weapon: API Surface

For JSON endpoints — API-mode apps, `/api` namespaces, or any controller that does `render json:`. The findings unique to this weapon are *data-shape* problems: Brakeman parses code, not payloads, and will never tell you a serializer ships a column it shouldn't.

## Over-exposure — the API-side IDOR

- **`render json: @user` (or `.to_json` without `only:`/serializer)** ships every column: token digests, role flags, internal ids, PII. Read the model's schema next to the render call and name the columns that leak. This is the single highest-yield check in this weapon.
- Serializers/jbuilder views with one shape for every consumer — admin-only fields delivered to regular users (pairs with `arsenal/authorization.md`).
- Associations included wholesale (`include:`, nested serializers) — the leak hides one level deep.
- Index endpoints with **no pagination**: `Model.all.to_json` is a table dump — data exposure and a self-DoS in one line.
- **Inertia props serialize models straight into the page**: `render inertia: "Show", props: { post: post.as_json }` (no `only:`) ships every column into the page's props — the same over-exposure as `render json:`, just rendered into the HTML/JS instead of a JSON body. Allowlist fields the same way (a model dumped into a `data-*` attribute via `to_json` leaks identically).

```ruby
# Problem — ships every column: password_digest, api_token, the admin flag, internal ids, PII
render json: User.find(params[:id])
render inertia: "Posts/Show", props: { post: Post.find(params[:id]).as_json }   # same leak, into the page props

# Fix — an explicit allowlist of fields (a serializer, or only:)
render json: User.find(params[:id]).as_json(only: [:id, :name, :avatar_url])
render inertia: "Posts/Show", props: { post: Post.find(params[:id]).as_json(only: %i[id title]) }
```

## Cross-origin and abuse

- **CORS**: `rack-cors` with `origins '*'` *combined with* `credentials: true`, or code reflecting the request's `Origin` header back. Wildcard without credentials is often fine for a public API — confirm intent before flagging.
- **No rate limiting** (rack-attack absent) on auth endpoints, token issuance, and expensive queries — pairs with the brute-force checks in `arsenal/authentication.md`.
- **ReDoS / user-controlled regex** — a regex with nested or overlapping quantifiers (`/^(\w+)+$/`, `(a|a)*`, `(.*)*`) matched against user input backtracks catastrophically: one request pegs a CPU (a DoS scanners catch only unreliably). Worse, `Regexp.new(params[:pattern])` lets the user *supply the regex* — ReDoS plus a match-anything bypass. Applies anywhere input reaches a regex — `params`, search, a model `format:` validation, not just APIs.

```ruby
Regexp.new(params[:q]) =~ text   # Problem — user supplies the regex: ReDoS + match-anything
params[:q] =~ /^(\w+)+$/          # Problem — nested quantifier on user input → catastrophic backtracking
Regexp.timeout = 1               # Fix (Ruby 3.2+) — a global regex timeout; and never build a regex from input
```

```ruby
# Problem — wildcard origin with credentials: any site makes authenticated requests as the user
Rails.application.config.middleware.insert_before 0, Rack::Cors do
  allow { origins "*"; resource "*", headers: :any, credentials: true }
end

# Fix — an explicit origin allowlist (never combine "*" with credentials)
allow { origins "https://app.example.com"; resource "/api/*", headers: :any, credentials: true }
```

## Tokens and errors

- API tokens accepted via **query string** — they end up in access logs, browser history, and Referer headers. Headers only.
- Error responses that serialize exceptions (`render json: { error: e.message }` with internals, or full backtraces in production JSON).

```ruby
# Problem — API token in the query string: it lands in access logs, history, and Referer headers
# GET /api/orders?api_token=secret123
token = params[:api_token]

# Fix — accept it only from a header
token = request.headers["Authorization"]&.delete_prefix("Bearer ")
```

## Inbound webhooks (Stripe, GitHub, Twilio, payment callbacks)

A webhook receiver is an unauthenticated public endpoint that performs privileged work — marks invoices paid, provisions accounts, deletes data. Being "internal" or unguessable is not authentication; the sender signs each payload and the receiver **must** verify that signature. This is where Brakeman is blind (it sees a controller, not a missing HMAC check) and where vibecoded apps reliably skip the step.

- **No signature verification at all** — the action reads `params` and acts on them with no check against a signing secret. Anyone who finds the URL can forge `payment.succeeded`. Confirmed-critical when the action touches money, provisioning, or deletion.
- **Verification not timing-safe** — `signature == computed` (or `==` on the HMAC) leaks the secret to a timing attack. The remedy is `ActiveSupport::SecurityUtils.secure_compare` (or the SDK's own verifier, e.g. `Stripe::Webhook.construct_event`).
- **HMAC computed over the parsed body, not the raw bytes.** Re-serializing `params.to_json` and signing that won't match the sender's signature over the raw request body — apps that "fix" the resulting mismatch by disabling verification end up with check-shaped code that verifies nothing. Confirm the controller reads `request.raw_post`/`request.body.read` for the digest.
- **No replay protection** on sensitive webhooks — a captured-and-replayed valid request re-triggers the side effect. Look for a timestamp-tolerance check and/or event-id idempotency (pairs with the idempotency note below).
- **CSRF skipped without signature verification put in its place** — `skip_before_action :verify_authenticity_token` on the webhook is correct *only* if signature verification replaces it. Skipped-and-unverified is the hole (cross-check `arsenal/hardening.md`).
- **Non-idempotent processing** — webhooks are delivered at-least-once; an action that double-charges or double-provisions on redelivery is a correctness-and-money bug. Look for dedup on the provider's event id.

```ruby
# Problem — acts on the payload with no signature check: anyone can forge "payment succeeded"
def stripe
  event = JSON.parse(request.body.read)
  Order.find(event["order_id"]).paid! if event["type"] == "payment_intent.succeeded"
end

# Fix — verify the signature over the raw body before trusting anything
def stripe
  event = Stripe::Webhook.construct_event(
    request.body.read, request.headers["Stripe-Signature"], ENV.fetch("STRIPE_WEBHOOK_SECRET")
  )   # raises on a forged or replayed-out-of-tolerance signature
end
```

## Outbound webhooks (if the app sends user-configured webhooks)

- **Editable destination URL** — if a user can change a webhook's destination *after* creation, they can repoint it at their own server and receive every future signed event payload (other users'/tenant data), with the existing secret authenticating the delivery. Make the destination immutable after create (retargeting = a new webhook + a new secret), and SSRF-validate it at send time, not just on create (see `arsenal/hardening.md`).

```ruby
params.expect(webhook: [:name, :url])  # Problem (on update) — repointing exfiltrates signed payloads
params.expect(webhook: [:name])        # Fix — destination immutable after create
```

## GraphQL (if the app uses graphql-ruby)

A GraphQL endpoint is one URL with a different threat model — none of the REST checks above catch it.

- **Introspection enabled in production** — the schema is a map of the whole attack surface; disable it outside development.
- **No query depth/complexity limit** — a deeply nested or expensive query is a one-request DoS. Look for `max_depth`/`max_complexity` on the schema; their absence is the finding.
- **No persisted-query allowlist** on a public-facing API where the client set is known.
- Authorization checked at the controller but not per-field/resolver — GraphQL bypasses controller-level guards; the IDOR/authorization weapon applies at the resolver level.

```ruby
# Problem — introspection on in production maps the whole attack surface; no depth limit = one-request DoS
class AppSchema < GraphQL::Schema
  # defaults: introspection enabled, no max_depth / max_complexity
end

# Fix — disable introspection outside dev and bound query cost
class AppSchema < GraphQL::Schema
  disable_introspection_entry_points unless Rails.env.development?
  max_depth 12
  max_complexity 300
end
```

## XML parsing (if the app accepts or parses XML)

- **XXE** — `Nokogiri`/`LibXML` parsing user XML without disabling DTD and external entities (`Nokogiri::XML(io)` with `NONET`/`NOENT` misconfigured) reads local files or hits internal URLs.
- **Entity expansion (Billion Laughs)** — no limit on nested entity expansion is a memory-exhaustion DoS. Disable entity expansion on user-supplied XML.

```ruby
# Problem — parsing user XML with entity substitution on → XXE: reads local files, hits internal URLs
doc = Nokogiri::XML(params[:xml]) { |c| c.noent }   # NOENT resolves external entities

# Fix — keep entity substitution off (the default) and block network access via DTD
doc = Nokogiri::XML(params[:xml]) { |c| c.nonet }
```

## OAuth flows (if the app is a provider via Doorkeeper or a client via OmniAuth)

- **`redirect_uri` not validated against a server-side allowlist** — open redirect that leaks the auth code/token. The single most common OAuth flaw.
- **Missing `state` parameter** in the client flow — CSRF on the authorization callback.
- **Scope not validated/defaulted** per application — over-broad grants.

```ruby
# Problem — redirect_uri echoed back without an allowlist → open redirect leaks the auth code/token
redirect_to params[:redirect_uri]

# Fix — validate against the application's registered URIs (a server-side allowlist)
uri = params[:redirect_uri]
redirect_to oauth_app.redirect_uris.include?(uri) ? uri : root_url
```

## Severity and remedies

A serializer leaking credentials or PII at scale competes with IDOR at the top of the report — state what leaks and to whom. CORS wildcard-with-credentials and unverified webhooks rank with the hardening tier unless a concrete abuse path is articulated. Remedies: explicit serializers with allowlisted fields, pagination defaults, rack-cors origin lists, rack-attack throttles, `secure_compare` on webhook signatures.
