<div align="center">

### One command. Every risk in your Rails app, ranked by impact.

<img src="assets/nuke-on-rails.png" alt="Nuke on Rails: a steam train hauling a nuclear bomb through the desert as a mushroom cloud erupts" width="100%">

<a href="https://twitter.com/alanalvestech">
  <img src="https://img.shields.io/badge/Follow on X-000000?style=for-the-badge&logo=x&logoColor=white" alt="Follow on X" />
</a>
<a href="https://www.linkedin.com/in/alanalvestech/">
  <img src="https://img.shields.io/badge/Follow on LinkedIn-0077B5?style=for-the-badge&logo=linkedin&logoColor=white" alt="Follow on LinkedIn" />
</a>
<a href="LICENSE">
  <img src="https://img.shields.io/badge/LICENSE-MIT-2ea44f?style=for-the-badge" alt="MIT License" />
</a>

</div>

<br />

<p align="center">
  <a href="#what-it-is">What It Is</a> &nbsp;&bull;&nbsp;
  <a href="#what-it-catches">What It Catches</a> &nbsp;&bull;&nbsp;
  <a href="#how-it-works">How It Works</a> &nbsp;&bull;&nbsp;
  <a href="#quick-start">Quick Start</a>
</p>

---

## What it is

**Nuke on Rails** is an open-source **skill** for AI coding agents (Claude Code, Cursor, Codex, and more), not a gem you add to your Gemfile. It audits your Rails app the way a principal engineer would: *what to refactor, what's vulnerable, and in what order to fix it.*

Instead of stapling separate tool reports together, it returns **a single list, ranked by impact**. An IDOR in your payments controller outranks a fat model; a high-churn fat model outranks a theoretical warning.

Scanners list problems. Nuke on Rails decides the order.

## What it catches

Coverage maps to the **OWASP Top 10 2025**. Each area is a [lens](lenses/): a plain-markdown check the audit applies on top of the scanners.

**[Access control & IDOR](lenses/authorization.md)**
- Records loaded by id without ownership scoping (the canonical payments / orders / invoices case)
- Authorization missing where authentication exists (logged-in is not allowed-to)
- Mass assignment: `permit!`, role escalation, nested attributes, raw-Hash bypass
- Records leaked through form dropdowns and serializers
- Cross-tenant leaks in multi-tenant apps; routes exposing actions that shouldn't be public

**[Authentication & sessions](lenses/authentication.md)**
- Devise misconfig: user enumeration, no lockout, sessions that never expire, weak password policy
- Session fixation and missing cookie flags (`secure` / `httponly` / `SameSite`)
- Timing attacks and type-juggling on token and credential lookups
- Tokens stored in plaintext or without expiry; rate-limit / throttle bypass
- Custom Warden strategy bugs, scope confusion, impersonation gaps; JWT pitfalls (`none` alg, no expiry)

**[Secrets](lenses/secrets.md)**
- `master.key`, credentials keys, or `.env` committed to git (including in history)
- Hardcoded API keys (Stripe, AWS, Twilio…) in code and initializers
- Secrets in seeds, fixtures, or `database.yml`; secret-as-ENV-fallback

**[Cryptography](lenses/cryptography.md)**
- Encryption oracles (one crypto routine reused for trust tokens and user data)
- Hand-rolled crypto instead of Rails primitives; static IVs; unauthenticated cipher modes
- Weak password hashing (MD5/SHA); sensitive columns (CPF, SSN, bank, health) stored in plaintext

**[Configuration & hardening](lenses/hardening.md)**
- `force_ssl` / HSTS off; backing-service traffic (Postgres, Redis) in cleartext
- CSP missing or disabled; CSRF skipped on cookie-authenticated actions; host-header injection
- Unauthenticated mounted dashboards (Sidekiq, PgHero, Flipper)
- Debug / console gems shipped to production (a remote-code-execution surface)
- Stack traces to users, unsafe uploads, stored XSS via markdown rendering

**[API surface](lenses/api.md)**
- JSON over-exposure (`render json:` leaking token digests, role flags, PII)
- Missing pagination (table dump and self-DoS); CORS wildcard with credentials; tokens in query strings
- Exception leakage; unverified webhooks
- GraphQL introspection and unbounded query depth/complexity; XXE and entity expansion; OAuth `redirect_uri` / `state` / scope flaws

**[Logging & monitoring](lenses/logging.md)**
- Sensitive data in logs (filter gaps, `puts` / logger dumps, unscrubbed error-tracker breadcrumbs)
- PII sent to third-party and LLM calls
- No audit trail on login, payment, privilege, and admin actions

**[Dependencies & versions](lenses/cve.md)**
- Known CVEs in your gems and in the Ruby version itself
- JavaScript dependency advisories; insecure or unpinned gem sources
- End-of-life Ruby or Rails (a critical compliance finding even with zero open CVEs)

**[Code quality](lenses/code-quality.md)**
- Fat models, callback-driven workflows, rug concerns, spaghetti branching, N+1, the churn × complexity hotspots

The community grows the catalog: a new check is a markdown PR, no code required.

## How it works

Deterministic scanners do the scanning; the LLM is the judge, not the author. On every run the skill:

1. **Detects** the project: full Rails app, plain Ruby (graceful degradation), or neither.
2. **Runs the scanners** and reads their machine-readable output. It brings its own tools and never touches your Gemfile.
3. **Picks the hotspots** by churn × complexity, reading deeply where it matters instead of reviewing everything uniformly.
4. **Triages**: it kills false positives by reading the actual code path, applies the lenses above, and adversarially verifies every security finding before it reaches the report. Anything it can't justify is downgraded to "theoretical," not sold as confirmed.
5. **Returns one report, ranked by impact** — a plan a principal engineer would sign, not a tool dump.

## Quick Start

Nuke on Rails ships through the [`skills`](https://skills.sh) CLI. You'll need [Node.js](https://nodejs.org).

**1. Install the `skills` CLI:**

```sh
npm install -g skills
```

**2. Add Nuke on Rails** (from your project root):

```sh
skills add nuke-on-rails/nuke-on-rails
```

It works across agents: Claude Code, Cursor, Codex, Gemini CLI, Warp, and more.

**3. Run it** inside your agent:

```
/nuke-on-rails
```

Zero setup beyond that. It installs its own tools, detects Rails vs. plain Ruby, runs everything, and hands you the plan. It never touches your Gemfile.

**4. Update** when you want the latest lenses and fixes:

```sh
skills update nuke-on-rails
```

## Star History

[![Star History Chart](https://api.star-history.com/svg?repos=nuke-on-rails/nuke-on-rails&type=Date)](https://star-history.com/#nuke-on-rails/nuke-on-rails&Date)
