# CLAUDE.md

Guidance for Claude Code (and other coding agents) working in this repository.

## What this project is

Nuke on Rails is an open source skill (Claude Code and cross-agent) that audits the health of a Rails project — the review a principal engineer would do: *what to refactor, what's vulnerable, and in what order to attack it*. It runs three deterministic engines, uses the LLM as the judge, and produces **one single list prioritized by impact**.

**The quality bar:** the skill must work first time on any Rails repo, with zero setup, and produce a result the developer couldn't get by just asking an agent to "review my code".

## Hard rules

- **Everything public is written in fluent English** — README, SKILL.md, lenses, generated reports, issues, PRs, releases, commit messages. The audience is the global Rails community.
- **Naming:** "Nuke on Rails" as the brand in prose; `nuke-on-rails` (kebab-case) in code, paths and commands; never `nukeonrails`.
- **Security findings are held to a higher bar than quality findings.** A weak quality finding gets ignored; a false security claim in a report burns trust. Every security finding must survive adversarial verification before reaching the report.

## Architecture

Three deterministic engines + LLM as judge:

1. **rubycritic** — quality. The churn × complexity quadrant decides where the LLM spends its context. Never review a codebase uniformly.
2. **Brakeman** — security. Brakeman is the engine; the LLM is the triager: filter false positives, explain the real exploit path. The LLM also *covers* (not "guarantees") what Brakeman can't reach: IDOR, authorization, business logic.
3. **bundler-audit** (+ **ruby_audit**) — known CVEs in gems and in the Ruby version itself.

## Design principles

- **Zero-dependency:** the skill installs the engines itself, detects Rails vs. plain Ruby, and degrades gracefully (plain Ruby: rubycritic + bundler-audit run, Brakeman is skipped).
- **One impact-ranked report**, never tool sections stapled together. An IDOR in a payments controller outranks a fat model; a high-churn fat model outranks a theoretical warning.
- **Lenses are plain markdown** (`lenses/code-quality.md`, `lenses/authorization.md`, `lenses/authentication.md`, …). The community contributes checks via text-only PRs; the maintainer owns the engine (the RuboCop/cops model).
- **The canonical source for security lenses is the official Rails Security Guide** (https://guides.rubyonrails.org/security.html) — distill from it, link findings to its sections, and re-check lenses against it on new Rails releases. Prefer it over community checklists, which go stale.

## Internal docs

`HANDOFF.md` (gitignored, in Portuguese) holds internal project context for the maintainer. Never quote or copy from it into public artifacts.
