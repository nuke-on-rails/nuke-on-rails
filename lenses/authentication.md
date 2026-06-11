# Lens: Authentication

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
- **`:timeoutable` not enabled**: sessions never expire.
- **`config.password_length` lowered**, or `:validatable` removed without a replacement policy.
- **`secret_key` / `pepper` hardcoded** in the initializer instead of credentials/ENV.
- For apps with compliance-ish needs (password expiry, blocking password reuse, limiting concurrent sessions): the concrete remedy is **devise-security** and its modules (`password_expirable`, `password_archivable`, `session_limitable`) — recommend the gem, don't let anyone hand-roll those.

## Custom Warden strategies and token flows

Custom strategies (API token login, impersonation, SSO callbacks) hide in `config/initializers/`. Classic bugs:

- **Token comparison with `==`** instead of `ActiveSupport::SecurityUtils.secure_compare` — timing attack. Worse: `where(token: params[:token])` looks the secret up *by value* — that's a timing oracle in the database. Look up by a non-secret id, then `secure_compare` the token.
- **Tokens stored in plain text columns** and/or without expiry; a DB read becomes account takeover. Store a digest, like password hashes.
- **`success!` on a path that should `fail!`**, or rescue blocks inside the strategy that swallow verification errors into a pass.
- **Warden scope confusion**: `warden.set_user(user)` on the wrong scope (e.g. granting the `:admin` scope from a user-level flow); impersonation features that never check the impersonator's privilege.
- **Failure apps / rescue handlers that leak *why* auth failed** ("wrong password" vs "no such user") — enumeration again.

## Sessions and resets

- **Session fixation**: `reset_session` missing on login and logout (Devise handles this; hand-rolled auth usually forgets).
- **Cookie flags**: session cookie without `secure`/`httponly`/SameSite in production config.
- **Sensitive data in the cookie session store** (role flags, feature gates the client can replay).
- **Password reset flow** (hand-rolled): token guessable or unexpired, token not invalidated after use, response revealing account existence.
- **JWTs, if present**: no expiry, `none` algorithm accepted, secret shared with other purposes, tokens irrevocable by design with no denylist.

## Severity

A confirmed flaw in authentication is account takeover or enumeration at scale — it competes with confirmed IDOR for the top of the report. Configuration weaknesses (paranoid off, no lockable) rank as hardening: above quality findings, below demonstrated exploits. As always: no articulated exploit path → "theoretical", never "confirmed".

## Preferred Remedies

- Point to the specific Devise module or config line — these fixes are usually one line in `devise.rb`.
- `secure_compare` for every secret comparison; digests for every stored token.
- `rack-attack` for throttling login and reset endpoints.
- devise-security for password policy / session-limit requirements.
- A request spec per auth rule: lockout locks, reset tokens expire, logout resets the session.
