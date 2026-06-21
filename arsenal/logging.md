# Weapon: Logging & Monitoring Failures

OWASP 2025 A09 — the one Top 10 category no engine and no other weapon covers, and the one RailsGoat itself barely plants because it's a failure of *absence*. Two opposite failure modes: too little logging (you can't tell you were breached) and too much (the logs themselves become the leak). Read `config/environments/production.rb`, `config/initializers/filter_parameter_logging.rb`, the auth and payment controllers, and the Gemfile (for an audit-trail gem).

Reference: Rails Security Guide §Logging and OWASP A09.

## Sensitive data in logs (the leak)

- **`config.filter_parameters` missing the app's real secrets.** Rails filters `:password` by default, but not tokens, API keys, `:ssn`, `:bank_account_num`, card data, `:otp`, or auth headers — those land in plaintext in production logs. Read the models' sensitive columns and confirm each is filtered. This is the highest-yield check here.
- **Manual `Rails.logger.info`/`puts`/`p` dumping params, user objects, or full request bodies** in controllers and jobs — bypasses parameter filtering entirely.
- **Full exception objects logged with their data**, or sensitive values in error-tracking breadcrumbs (Sentry/Rollbar) without scrubbing. The concrete, grep-able form is `Sentry.init` with `send_default_pii = true` (or unset), which auto-attaches the user's email/IP/name and the request cookies to every error sent to a third party.

  ```ruby
  Sentry.init { |c| c.send_default_pii = true }   # Problem — emails, IPs, cookies auto-sent to a third party
  Sentry.init do |c|                              # Fix — keep PII off; strip cookies before send
    c.send_default_pii = false
    c.before_send = ->(event, _hint) { event.request&.cookies = {}; event }
  end
  ```
- **Unredacted user data sent to third-party services — especially LLMs.** A controller or job that pipes `params[:message]`, a user record, or free text straight into an OpenAI/Anthropic/external API call leaks PII outside the trust boundary. This is a signature flaw of AI-built apps (they integrate AI casually) and no engine sees it. Trace user data into outbound HTTP/SDK calls; the named remedy is redacting PII first (e.g. the `top_secret` gem: filter before the call, restore after).
- **Tokens/session ids logged** by custom middleware or `before_action` debug code left in.

```ruby
# Problem — Rails filters :password by default, but not these — they land in plaintext prod logs
# config/initializers/filter_parameter_logging.rb
Rails.application.config.filter_parameters += [:password]   # ssn, card_number, api_token NOT filtered
# ...and an explicit dump bypasses parameter filtering entirely:
Rails.logger.info "charge: #{params.to_unsafe_h}"

# Fix — filter every sensitive field (a regex catches variants); never dump raw params
Rails.application.config.filter_parameters += [
  :password, :ssn, :card_number, :cvv, :otp, /token/, /secret/, /_key/
]
```

## Missing security logging (the blind spot)

- **No audit trail on security-critical events**: login success/failure, password/email change, privilege change, payment, admin actions. If an attacker can act and nothing records it, detection and forensics are impossible. Flag the absence on money/account paths specifically.
- **Audit trail destroyed with the record it tracks** — an audit/version log wired with `dependent: :destroy` (or an `audited`/`paper_trail`/`phi_access_logs` table cascade-deleted with its parent) vanishes exactly when forensics need it: after the record is gone, including a malicious deletion. Audit logs must outlive what they track (HIPAA mandates 6-year retention; every app needs them after an incident). Keep them out of the destroy cascade; prefer an append-only store.

  ```ruby
  has_many :access_logs, dependent: :destroy  # Problem — deleting the patient wipes its audit trail
  has_many :access_logs                        # Fix — logs survive; never cascade-delete an audit trail
  ```
- **Failed-authentication attempts not recorded** — no way to see credential stuffing in progress (pairs with the brute-force checks in `arsenal/authentication.md` and rack-attack).
- **No distinction in logs between a user acting on their own data and on someone else's** — makes IDOR exploitation invisible after the fact.

```ruby
# Problem — money changes hands and nothing records who did it → no forensics after an incident
def refund
  order.refund!(params[:amount])   # actor, target, amount, time all unrecorded
  redirect_to order
end

# Fix — record security-critical events (audited / paper_trail, or an explicit audit log)
def refund
  order.refund!(params[:amount])
  AuditLog.create!(actor: current_user, action: "refund", target: order, amount: params[:amount])
end
```

## Severity and remedies

Sensitive data in logs is the confirmable finding here — state what leaks and where (prod logs, error tracker). Rank it with the hardening tier, or higher if it's credentials/PII at volume. Missing audit logging is a real but lower-severity design finding — frame it as "you would not detect this attack", not as an exploit. Remedies: add the fields to `filter_parameters` (and Logstop as a catch-all net for sensitive data in logs); scrub error-tracker payloads; add structured audit logging (`audited`/`paper_trail`) on the security-critical events; record auth attempts with Authtrail when the app uses Devise. Keep the report honest — absence-of-logging is "you're blind here", never "you're exploited here".
