# Lens: Committed Secrets

API keys pasted into code and key files committed to git — alongside IDOR, the signature mistake of unreviewed AI-generated apps. No engine in the pipeline covers this; the highest-signal checks are deterministic and need no extra tooling.

## Deterministic checks (run these, don't guess)

```sh
git ls-files | grep -E 'master\.key|credentials.*\.key|\.env($|\.)'   # key files under version control
git log --diff-filter=D --name-only --format= | sort -u | grep -E 'master\.key|\.env'   # deleted ≠ gone
```

- **`config/master.key` (or any `credentials/*.key`) committed** — `credentials.yml.enc` is now plaintext for anyone with repo access. Critical.
- **`.env` / `.env.production` versioned** — same class, no decryption needed.
- A key file that was committed once and later deleted is **still compromised**: it lives in history.

## Code reading (on initializers, config, and hotspot files)

- Hardcoded credentials: `sk_live_`/`sk-` (Stripe/OpenAI), `AKIA` (AWS), Twilio SIDs, SMTP passwords, JWT secrets, webhook signing secrets — especially in `config/initializers/` and service classes.
- Devise `pepper`/`secret_key` inline (the authentication lens checks this too).
- Secrets in seeds, fixtures, or `database.yml` with production credentials.
- `ENV.fetch("X", "actual-secret-as-fallback")` — the fallback ships the secret.

```ruby
# Problem — a live secret pasted into code (now in git history forever, even if deleted later)
STRIPE = Stripe::Client.new("sk_live_51H8xQ2eZvKYlo3kQ...")
secret = ENV.fetch("JWT_SECRET", "super-secret-prod-value")   # the fallback ships the secret

# Fix — read from credentials / ENV, keep the secret out of the repo, then ROTATE the leaked one
STRIPE = Stripe::Client.new(Rails.application.credentials.dig(:stripe, :secret_key))
secret = ENV.fetch("JWT_SECRET")   # no fallback — fail loudly if it's unset
```

## Severity and reporting

A live credential in the repo is a **confirmed critical finding** — it competes with IDOR for the top of the report. State the blast radius ("this Stripe live key can issue refunds"). A key in history but rotated since, or clearly a test/dummy key, goes to "theoretical" — verify before accusing; a false "your key is leaked" burns trust like any false security claim.

## Preferred Remedies

- **Rotate first.** Removing the file or rewriting history does not un-leak a key. Rotation is step one, always.
- Move secrets to Rails credentials (`rails credentials:edit`) or real ENV injection; keep `master.key` and `.env` in `.gitignore`.
- Replace committed `.env` with a `.env.example` of empty placeholders.
- Add a preventive pre-commit guard so it can't happen again: git-secrets or gitleaks.
