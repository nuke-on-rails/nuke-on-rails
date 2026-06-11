# Lens: API Surface

For JSON endpoints — API-mode apps, `/api` namespaces, or any controller that does `render json:`. The findings unique to this lens are *data-shape* problems: Brakeman parses code, not payloads, and will never tell you a serializer ships a column it shouldn't.

## Over-exposure — the API-side IDOR

- **`render json: @user` (or `.to_json` without `only:`/serializer)** ships every column: token digests, role flags, internal ids, PII. Read the model's schema next to the render call and name the columns that leak. This is the single highest-yield check in this lens.
- Serializers/jbuilder views with one shape for every consumer — admin-only fields delivered to regular users (pairs with `lenses/authorization.md`).
- Associations included wholesale (`include:`, nested serializers) — the leak hides one level deep.
- Index endpoints with **no pagination**: `Model.all.to_json` is a table dump — data exposure and a self-DoS in one line.

## Cross-origin and abuse

- **CORS**: `rack-cors` with `origins '*'` *combined with* `credentials: true`, or code reflecting the request's `Origin` header back. Wildcard without credentials is often fine for a public API — confirm intent before flagging.
- **No rate limiting** (rack-attack absent) on auth endpoints, token issuance, and expensive queries — pairs with the brute-force checks in `lenses/authentication.md`.

## Tokens and errors

- API tokens accepted via **query string** — they end up in access logs, browser history, and Referer headers. Headers only.
- Error responses that serialize exceptions (`render json: { error: e.message }` with internals, or full backtraces in production JSON).
- Webhook endpoints without signature verification — being "internal" or unguessable is not authentication.

## Severity and remedies

A serializer leaking credentials or PII at scale competes with IDOR at the top of the report — state what leaks and to whom. CORS wildcard-with-credentials and unverified webhooks rank with the hardening tier unless a concrete abuse path is articulated. Remedies: explicit serializers with allowlisted fields, pagination defaults, rack-cors origin lists, rack-attack throttles, `secure_compare` on webhook signatures.
