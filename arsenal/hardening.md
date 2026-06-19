# Lens: Web Hardening

Production configuration the Rails Security Guide mandates and no engine meaningfully checks. The files to read are few and cheap: `config/environments/production.rb`, `config/initializers/` (CSP, session store, filter parameters), `config/routes.rb` (mounted dashboards), `ApplicationController`. A vibecoded app ships with whatever the generator left — including everything that was generated *commented out*.

Reference: Rails Security Guide — link findings to its sections (https://guides.rubyonrails.org/security.html).

## Transport and cookies

- **`config.force_ssl` not `true` in production** — session cookies travel over HTTP; HSTS absent. One line, high impact. (Behind a proxy, check `assume_ssl` too.)
- **Backing-service connections in cleartext** — read `config/database.yml`, `config/cable.yml`, and Redis/cache config: Postgres without `sslmode: verify-full`, MySQL without `VERIFY_IDENTITY`, Redis without `ssl: true`. App↔database traffic in cleartext leaks every row in transit; visible in config, missed by every engine.
- **Session cookie flags**: `secure`, `httponly`, and an explicit `SameSite` on the session store config. Cross-check with `arsenal/authentication.md` for fixation/store issues.

## Headers and CSP

- **Content-Security-Policy**: Rails generates `config/initializers/content_security_policy.rb` *commented out* — the most common state is "exists, disabled". Flag CSP absent, or stuck in `report_only` with nobody reading reports.
- Default header set (X-Frame-Options, X-Content-Type-Options) is fine out of the box — flag code that *removes* defaults.

## CSRF and hosts

- `protect_from_forgery` skipped (`skip_before_action :verify_authenticity_token`) with a broad `only:`/`except:` list — legitimate for token-authenticated APIs, a hole for cookie-authenticated actions.
- **Host authorization** (`config.hosts`) cleared or wildcarded in production — DNS-rebinding protection off.

## Exposure through config

- **Unauthenticated mounted dashboards** — `mount Sidekiq::Web`, PgHero, Flipper UI in `routes.rb` without an authentication constraint: confirmed-critical territory (job args often contain PII; some dashboards can execute things), not a config nit.
- **Debug/console gems reachable in production** — `web-console`, `better_errors`, `binding_of_caller` in the default or `production` Gemfile group (not confined to `:development`), or `config.web_console.*` left enabled: an exposed console is remote code execution. Read the Gemfile groups and the production config. Confirmed-critical.
- `config.consider_all_requests_local = true` in production — stack traces to users.
- `config.filter_parameters` missing the app's actual sensitive fields (tokens, documents, card data) — secrets in logs.

## Browser and proxy caching of authenticated content

Sensitive pages (account, payment, anything behind login) cached client-side or by an intermediary leak after logout or on shared machines. No engine checks this.

- Sensitive responses without `Cache-Control: no-store, private` (Rails doesn't set this by default for authenticated pages).
- Token-bearing URLs (password reset, email confirmation, magic links) without `<meta name="robots" content="noindex, nofollow">` — they get indexed and land in search results / referrer logs.
- `autocomplete="off"` missing on sensitive form fields (payment, SSN) where browser storage is a real exposure.
- `config.action_controller.default_url_options` host unset (host-header injection in generated links) and `asset_host` unset (cache-poisoning surface).

## Files

- Uploads: no content-type/extension allowlist, or user-uploaded HTML/SVG served from the app's own domain (stored XSS). ActiveStorage/CarrierWave validations are semantic — Brakeman won't see their absence.
- `send_file` / `send_data` with user-influenced paths or filenames (Brakeman flags some; confirm the rest).
- **User-supplied markdown/HTML rendered unsafely** — `Redcarpet::Render::HTML` (instead of `::Safe`) on user content, `filter_html: false`, CommonMarker without `:SAFE`, or `sanitize` with an over-broad tag allowlist: stored XSS that Brakeman doesn't reliably catch at the library-config level.

## Severity and remedies

Most findings here are the **hardening tier**: above quality findings, below demonstrated exploits. Two exceptions that rank as confirmed-critical when present: an unauthenticated Sidekiq/admin dashboard, and a cookie-authenticated action with CSRF skipped. Remedies are almost always one line in `production.rb` or `routes.rb` — say which line. For throttling, the named remedy is rack-attack.
