# Weapon: Authentication

How the app knows who you are — and how much that mechanism can be trusted. Brakeman checks a few auth-adjacent patterns; the configuration choices, custom strategies, and token flows below are semantic territory only code-reading covers.

## Step 1 — Identify the auth stack

The stack determines how suspicious to be. In rising order of scrutiny:

1. **Devise** — battle-tested defaults; the risk is in the *configuration* (below).
2. **Rodauth / authentication-zero generated code** — solid bases; check what was edited after generation.
3. **`has_secure_password` hand-wired** — bcrypt is right, but everything around it (sessions, resets, throttling) is hand-rolled; read all of it.
4. **Raw Warden strategies or fully hand-rolled auth** — treat as a mandatory hotspot regardless of churn × complexity. In an unreviewed codebase, hand-rolled authentication is usually the most dangerous file in the repo.

## Devise configuration review

The default `devise.rb` of a generated app is rarely touched. Check:

- **`config.paranoid = false`** (the default): password-reset and confirmation responses reveal whether an e-mail exists — user enumeration. Recommend `paranoid = true`.
- **`:lockable` not enabled**: unlimited password attempts; combined with no rate limiting (`rack-attack` absent), credential stuffing runs free.
- **Throttle key not normalized like the login lookup**: if rack-attack throttles by raw `params[:email]` but auth downcases/strips before finding the user, an attacker varies case and whitespace (`Foo@x.com`, `foo@x.com `) to get a fresh throttle bucket per attempt — the rate limit is bypassed while still hitting one account. The throttle discriminator must normalize identically to the authentication path.
- **`:timeoutable` not enabled**: sessions never expire.
- **`config.password_length` lowered**, or `:validatable` removed without a replacement policy.
- **`secret_key` / `pepper` hardcoded** in the initializer instead of credentials/ENV.
- For apps with compliance-ish needs (password expiry, blocking password reuse, limiting concurrent sessions): the concrete remedy is **devise-security** and its modules (`password_expirable`, `password_archivable`, `session_limitable`) — recommend the gem, don't let anyone hand-roll those.

```ruby
# Problem — generated devise.rb left at defaults: user enumeration + a committed secret
config.paranoid = false                    # reset/confirmation replies reveal whether an email exists
config.pepper   = "hardcoded-pepper-value" # secret in the repo instead of credentials/ENV
# (and :lockable absent from the model → unlimited password attempts)

# Fix
config.paranoid = true                     # uniform responses, no enumeration
config.pepper   = Rails.application.credentials.devise_pepper
# add :lockable to the model and throttle login with rate_limit / rack-attack
```

## Custom Warden strategies and token flows

Custom strategies (API token login, impersonation, SSO callbacks) hide in `config/initializers/`. Classic bugs:

- **Token comparison with `==`** instead of `ActiveSupport::SecurityUtils.secure_compare` — timing attack. Worse: `where(token: params[:token])` looks the secret up *by value* — that's a timing oracle in the database. Look up by a non-secret id, then `secure_compare` the token.
- **Type confusion on secret/credential lookups.** `find_by(token: params[:user][:token])` fed a JSON `{"token": 0}` (or an array/hash) can bypass the check entirely — MySQL coerces the string column to a number and `0` matches any non-numeric token. Coerce the param to String before any secret lookup (`params[...].to_s`) and compare with `secure_compare`; never trust that a param is the scalar type you expect. Invisible to static analysis.
- **Tokens stored in plain text columns** and/or without expiry; a DB read becomes account takeover. Store a digest, like password hashes.
- **`success!` on a path that should `fail!`**, or rescue blocks inside the strategy that swallow verification errors into a pass.
- **Warden scope confusion**: `warden.set_user(user)` on the wrong scope (e.g. granting the `:admin` scope from a user-level flow); impersonation features that never check the impersonator's privilege.
- **Failure apps / rescue handlers that leak *why* auth failed** ("wrong password" vs "no such user") — enumeration again.

```ruby
# Problem — the secret is looked up by value, then compared with ==: a timing oracle twice over
user = User.find_by(api_token: params[:token])
sign_in(user) if user && user.api_token == params[:token]

# Fix — look it up by a non-secret id, then compare a stored digest in constant time
user = User.find_by(id: params[:user_id])
if user && ActiveSupport::SecurityUtils.secure_compare(
     user.api_token_digest, Digest::SHA256.hexdigest(params[:token].to_s))   # .to_s blocks type confusion
  sign_in(user)
end
```

## Sessions and resets

- **Session fixation**: `reset_session` missing on login and logout (Devise handles this; hand-rolled auth usually forgets).
- **Cookie flags**: session cookie without `secure`/`httponly`/SameSite in production config.
- **Sensitive data in the cookie session store** (role flags, feature gates the client can replay).
- **Open redirect after login**: a post-login redirect built from a user-controlled param — `redirect_to params[:return_to]`, or `after_sign_in_path_for` reading `params`/`session[:return_to]` — without an origin allowlist. An attacker sends a login link that bounces the authenticated user to a look-alike phishing site. Validate against an internal path/host allowlist; never redirect to a raw param. (The OAuth `redirect_uri` variant lives in `arsenal/api.md`.)
- **Password reset flow** (hand-rolled): token guessable or unexpired, token not invalidated after use, response revealing account existence.
- **JWTs, if present**: no expiry, `none` algorithm accepted, secret shared with other purposes, tokens irrevocable by design with no denylist, or **sensitive data/secrets in the payload** — a JWT is base64, not encrypted, so anyone holding the token reads every claim (roles, PII, another service's token). Cross-check `arsenal/secrets.md` and `arsenal/logging.md`.

```ruby
# Problem — hand-rolled login never rotates the session id → session fixation
def create
  user = User.authenticate_by(email: params[:email], password: params[:password])
  session[:user_id] = user.id if user   # the pre-login session id survives, so an attacker can fixate it
end

# Fix — reset the session on every privilege change (login and logout)
def create
  user = User.authenticate_by(email: params[:email], password: params[:password])
  if user
    reset_session
    session[:user_id] = user.id
  end
end
```

## Severity

A confirmed flaw in authentication is account takeover or enumeration at scale — it competes with confirmed IDOR for the top of the report. Configuration weaknesses (paranoid off, no lockable) rank as hardening: above quality findings, below demonstrated exploits. As always: no articulated exploit path → "theoretical", never "confirmed".

## Preferred Remedies

- Point to the specific Devise module or config line — these fixes are usually one line in `devise.rb`.
- `secure_compare` for every secret comparison; digests for every stored token. For password login, prefer Rails 7+ `User.authenticate_by(email:, password:)` — it's built to be timing-safe and resists enumeration, unlike a `find_by` + `authenticate` pair.
- For throttling login/reset endpoints, the Rails 7.2+ built-in `rate_limit to:, within:, only:` in the controller is the native option (the throttle-key normalization caveat above applies to it too); `rack-attack` when you need cross-cutting or IP-level rules.
- devise-security for password policy / session-limit requirements.
- A request spec per auth rule: lockout locks, reset tokens expire, logout resets the session.
