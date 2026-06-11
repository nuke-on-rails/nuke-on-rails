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
  <a href="#who-its-for">Who It's For</a> &nbsp;&bull;&nbsp;
  <a href="#how-it-works">How It Works</a> &nbsp;&bull;&nbsp;
  <a href="#quick-start">Quick Start</a>
</p>

---

## What it is

**Nuke on Rails** audits your Rails app the way a principal engineer would: it tells you *what to refactor, what's vulnerable, and in what order to fix it.* One open source skill (Claude Code and cross-agent) powered by three battle-tested engines and a catalog of lenses, with an LLM in the judge's seat.

Three engines, a stack of lenses, one verdict. Instead of stapling reports together, Nuke on Rails returns **a single list, ranked by impact**. An IDOR in your payments controller outranks a fat model; a high-churn fat model outranks a theoretical warning.

RuboCop, Brakeman and rubycritic **list** problems. Nuke on Rails **decides the order**.

## Who It's For

- **Rails developers** who want a principal-engineer-grade audit on demand.
- **Vibecoders.** If you built your Rails app with AI and can't fully review the code yourself, this is the safety check you didn't know you needed: mass assignment, missing authorization, IDOR, gems with known CVEs.

Plain Ruby projects (gems, CLIs) work too with graceful degradation: rubycritic and bundler-audit run, Brakeman is skipped.

## How It Works

Nuke on Rails runs three deterministic engines, widened by a catalog of **lenses** that reach what static analysis can't:

- **[Rubycritic](https://github.com/whitesmith/rubycritic)** for **code quality**. The churn × complexity quadrant decides where attention goes first.
- **[Brakeman](https://brakemanscanner.org/)** for **security**. The LLM triages every warning: kills false positives, explains the real exploit path.
- **[Bundler-audit](https://github.com/rubysec/bundler-audit)** + **[Ruby_audit](https://github.com/civisanalytics/ruby_audit)** for **known CVEs** in your gems and in the Ruby version itself.
- **[Authorization](lenses/authorization.md)** : IDOR, missing authorization, mass assignment, nested-attribute and form-helper leaks.
- **[Authentication](lenses/authentication.md)** : Devise misconfig, custom Warden strategies, session fixation, timing attacks, throttle bypass.
- **[Secrets](lenses/secrets.md)** : committed keys, hardcoded credentials, `.env` in version control.
- **[Cryptography](lenses/cryptography.md)** : encryption oracles, hand-rolled crypto, weak hashing, plaintext sensitive columns.
- **[Hardening](lenses/hardening.md)** : TLS and HSTS, CSP, CSRF config, unauthenticated mounted dashboards, backing-service TLS.
- **[API](lenses/api.md)** : JSON over-exposure, CORS, GraphQL depth and introspection, XXE, OAuth flows.
- **[Logging](lenses/logging.md)** : sensitive data in logs, missing audit trail on critical events, PII leaking into LLM prompts.
- **[CVE](lenses/cve.md)** : JavaScript deps, day-zero web cross-checks, end-of-life Ruby or Rails versions.
- **[Code-quality](lenses/code-quality.md)** : fat models, callback-driven logic, rug concerns, spaghetti branching. The thermo-nuclear quality bar, translated to Rails.

The first three are the deterministic engines; the rest are **lenses**, plain-markdown checks covering the OWASP Top 10 2025. The community grows the catalog: a new check is a text-only PR, no code required.

On every run, the skill:

1. **Detects** the project: full Rails app, plain Ruby (graceful degradation), or neither.
2. **Runs the engines** and captures machine-readable output. It brings its own tools and never touches your Gemfile.
3. **Picks the hotspots** with the churn × complexity quadrant, reading deeply where it matters instead of reviewing everything uniformly.
4. **Triages and applies the lenses.** You are the judge, not the scanner: every finding is adversarially verified before it reaches the report, and security findings are held to a higher bar than quality ones.
5. **Returns one impact-ranked report.** A plan a principal engineer would sign, not a tool dump.

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

> Don't want a global install? Prefix any command with `npx` instead: `npx skills add nuke-on-rails/nuke-on-rails`.

**3. Run it** inside your agent:

```
/nuke-on-rails
```

Zero setup beyond that. The skill installs its own engines (rubycritic, Brakeman, bundler-audit, ruby_audit), detects Rails vs. plain Ruby, runs everything, and hands you the plan. It never touches your Gemfile.

**4. Update** when you want the latest lenses and fixes:

```sh
skills update nuke-on-rails
```

## Design principles

- **One impact-ranked report**, not tool sections.
- **Adversarial verification** of every finding before it reaches the report. A false security claim is worse than a missed one.
- **Pluggable lenses in plain markdown.** Contribute a new check with a text PR, no code required.

## Star History

[![Star History Chart](https://api.star-history.com/svg?repos=nuke-on-rails/nuke-on-rails&type=Date)](https://star-history.com/#nuke-on-rails/nuke-on-rails&Date)
