# Lens: Web Hardening

Production configuration the Rails Security Guide mandates and no engine meaningfully checks. The files to read are few and cheap: `config/environments/production.rb`, `config/initializers/` (CSP, session store, filter parameters), `config/routes.rb` (mounted dashboards), `ApplicationController`. A vibecoded app ships with whatever the generator left — including everything that was generated *commented out*.

Reference: Rails Security Guide — link findings to its sections (https://guides.rubyonrails.org/security.html).

## Transport and cookies

- **`config.force_ssl` not `true` in production** — session cookies travel over HTTP; HSTS absent. One line, high impact. (Behind a proxy, check `assume_ssl` too.)
- **Backing-service connections in cleartext** — read `config/database.yml`, `config/cable.yml`, and Redis/cache config: Postgres without `sslmode: verify-full`, MySQL without `VERIFY_IDENTITY`, Redis without `ssl: true`. App↔database traffic in cleartext leaks every row in transit; visible in config, missed by every engine.
- **Session cookie flags**: `secure`, `httponly`, and an explicit `SameSite` on the session store config. Cross-check with `arsenal/authentication.md` for fixation/store issues.

```ruby
# Problem — production serves over HTTP: session cookies travel in cleartext, HSTS absent
# config/environments/production.rb
config.force_ssl = false   # (or left commented out by the generator)

# Fix — force TLS app-wide (sets Secure cookies + HSTS in one line)
config.force_ssl = true
```

## Headers and CSP

- **Content-Security-Policy**: Rails generates `config/initializers/content_security_policy.rb` *commented out* — the most common state is "exists, disabled". Flag CSP absent, or stuck in `report_only` with nobody reading reports.
- Default header set (X-Frame-Options, X-Content-Type-Options) is fine out of the box — flag code that *removes* defaults.

```ruby
# Problem — the generated CSP initializer ships entirely commented out: no policy at all
# config/initializers/content_security_policy.rb
# Rails.application.config.content_security_policy do |policy|
#   policy.default_src :self
# end

# Fix — enable a real policy (don't sit forever in report_only with nobody reading reports)
Rails.application.config.content_security_policy do |policy|
  policy.default_src :self
  policy.script_src  :self
end
```

## CSRF and hosts

- `protect_from_forgery` skipped (`skip_before_action :verify_authenticity_token`) with a broad `only:`/`except:` list — legitimate for token-authenticated APIs, a hole for cookie-authenticated actions.
- **Host authorization** (`config.hosts`) cleared or wildcarded in production — DNS-rebinding protection off.

```ruby
# Problem — CSRF protection disabled on a cookie-authenticated controller → forgeable state changes
class TransfersController < ApplicationController
  skip_before_action :verify_authenticity_token   # not a token-only API: this is a hole
end

# Fix — keep CSRF on for cookie-auth actions; skip only for genuinely token-authenticated APIs
class TransfersController < ApplicationController
  protect_from_forgery with: :exception
end
```

## Exposure through config

- **Unauthenticated mounted dashboards** — `mount Sidekiq::Web`, `mount MissionControl::Jobs::Engine` (the Solid Queue dashboard, default in Rails 8), PgHero, Flipper UI in `routes.rb` without an authentication constraint: confirmed-critical territory (job args often contain PII; some dashboards can retry/discard jobs or execute things), not a config nit. On a modern Rails 8 app the exposed dashboard is usually Mission Control, not Sidekiq — check for both.
- **Debug/console gems reachable in production** — `web-console`, `better_errors`, `binding_of_caller` in the default or `production` Gemfile group (not confined to `:development`), or `config.web_console.*` left enabled: an exposed console is remote code execution. Read the Gemfile groups and the production config. Confirmed-critical.
- `config.consider_all_requests_local = true` in production — stack traces to users.
- `config.filter_parameters` missing the app's actual sensitive fields (tokens, documents, card data) — secrets in logs.

```ruby
# Problem — Sidekiq's web UI mounted with no auth: job args (often PII) exposed to anyone who finds
# the path, and the dashboard can retry/kill jobs. config/routes.rb
mount Sidekiq::Web => "/sidekiq"

# Fix — gate it behind authentication (admins only)
authenticate :user, ->(u) { u.admin? } do
  mount Sidekiq::Web => "/sidekiq"
end
```

## Browser and proxy caching of authenticated content

Sensitive pages (account, payment, anything behind login) cached client-side or by an intermediary leak after logout or on shared machines. No engine checks this.

- Sensitive responses without `Cache-Control: no-store, private` (Rails doesn't set this by default for authenticated pages).
- Token-bearing URLs (password reset, email confirmation, magic links) without `<meta name="robots" content="noindex, nofollow">` — they get indexed and land in search results / referrer logs.
- `autocomplete="off"` missing on sensitive form fields (payment, SSN) where browser storage is a real exposure.
- `config.action_controller.default_url_options` host unset (host-header injection in generated links) and `asset_host` unset (cache-poisoning surface).

```ruby
# Problem — an authenticated page is cacheable: it lingers in the browser/proxy after logout
def show
  @statement = current_user.statements.find(params[:id])   # no cache directives → shared-machine leak
end

# Fix — mark sensitive responses uncacheable
def show
  @statement = current_user.statements.find(params[:id])
  response.headers["Cache-Control"] = "no-store, private"
end
```

## Files

- Uploads: no content-type/extension allowlist, or user-uploaded HTML/SVG served from the app's own domain (stored XSS). ActiveStorage/CarrierWave validations are semantic — Brakeman won't see their absence.
- `send_file` / `send_data` with user-influenced paths or filenames (Brakeman flags some; confirm the rest).
- **User-supplied markdown/HTML rendered unsafely** — `Redcarpet::Render::HTML` (instead of `::Safe`) on user content, `filter_html: false`, CommonMarker without `:SAFE`, or `sanitize` with an over-broad tag allowlist: stored XSS that Brakeman doesn't reliably catch at the library-config level.

```ruby
# Problem — no content-type allowlist: a user uploads .svg/.html served from your domain → stored XSS
class Document < ApplicationRecord
  has_one_attached :file   # accepts anything
end

# Fix — validate the content type (and serve user files from a separate domain)
class Document < ApplicationRecord
  has_one_attached :file
  validates :file, content_type: ["image/png", "image/jpeg", "application/pdf"]
end
```

## Severity and remedies

Most findings here are the **hardening tier**: above quality findings, below demonstrated exploits. Two exceptions that rank as confirmed-critical when present: an unauthenticated Sidekiq/admin dashboard, and a cookie-authenticated action with CSRF skipped. Remedies are almost always one line in `production.rb` or `routes.rb` — say which line. For throttling, the named remedy is rack-attack.
