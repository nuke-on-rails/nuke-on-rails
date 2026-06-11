<div align="center">

### One command. Every risk in your Rails app, ranked by impact.

<img src="assets/nuke-on-rails.png" alt="Nuke on Rails — a steam train hauling a nuclear bomb through the desert as a mushroom cloud erupts" width="100%">

[![License: MIT](https://img.shields.io/badge/LICENSE-MIT-22c55e?style=for-the-badge)](LICENSE)
[![Built for Claude Code](https://img.shields.io/badge/BUILT%20FOR-CLAUDE%20CODE-black?style=for-the-badge)](https://claude.com/claude-code)
[![Follow on X](https://img.shields.io/badge/FOLLOW%20ON-X-black?style=for-the-badge&logo=x)](https://x.com/)
[![Follow on LinkedIn](https://img.shields.io/badge/FOLLOW%20ON-LINKEDIN-0a66c2?style=for-the-badge&logo=linkedin)](https://www.linkedin.com/)

</div>

## What it is

A full project health audit for Rails apps — the review a principal engineer would do: *what to refactor, what's vulnerable, and in what order to attack it*.

Nuke on Rails is an open source skill (Claude Code and cross-agent) that runs three battle-tested engines and uses an LLM as the judge:

- **[rubycritic](https://github.com/whitesmith/rubycritic)** — code quality. The churn × complexity quadrant decides where attention goes first.
- **[Brakeman](https://brakemanscanner.org/)** — security. The LLM triages: filters false positives and explains the actual exploit path.
- **[bundler-audit](https://github.com/rubysec/bundler-audit)** + **[ruby_audit](https://github.com/civisanalytics/ruby_audit)** — known CVEs in your gems and in your Ruby version itself.

Instead of stapling three reports together, it produces **one single list, prioritized by impact** — an IDOR in your payments controller comes before a fat model; a high-churn fat model comes before a theoretical warning.

RuboCop, Brakeman and rubycritic **list** problems. Nuke on Rails **decides the order**.

## Who it's for

- **Rails developers** who want a principal-engineer-grade audit on demand.
- **Vibecoders** — if you built your Rails app with AI and can't fully review the code yourself, this is the safety check you didn't know you needed: mass assignment, missing authorization, IDOR, gems with known CVEs.

Plain Ruby projects (gems, CLIs) work too with graceful degradation: rubycritic and bundler-audit run, Brakeman is skipped.

## How it will work

```sh
npx skills add nuke-on-rails/nuke-on-rails
```

Then, inside your agent:

```
/nuke-on-rails
```

Zero setup — the skill installs the engines, detects Rails vs. plain Ruby, runs everything and hands you the plan.

## Design principles

- **One impact-ranked report**, not tool sections.
- **Adversarial verification** of every finding before it reaches the report — a false security claim is worse than a missed one.
- **Pluggable lenses in plain markdown** (`lenses/authorization.md`, `lenses/cryptography.md`, `lenses/logging.md`, …, covering the OWASP Top 10) — contribute a new check with a text PR, no code required.

## Credits

The name came from [Ton](https://github.com/devton), the project's first contributor.

Maintained by Alan.
