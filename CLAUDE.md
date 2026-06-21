# CLAUDE.md

Guidance for Claude Code (and other coding agents) working in this repository.

## What this project is

Nuke on Rails is an open source skill (Claude Code and cross-agent) that audits the health of a Rails project — the review a principal engineer would do: *what to refactor, what's vulnerable, and in what order to attack it*. It runs three deterministic engines, uses the LLM as the judge, and produces **one single list prioritized by impact**.

**The quality bar:** the skill must work first time on any Rails repo, with zero setup, and produce a result the developer couldn't get by just asking an agent to "review my code".

## Hard rules

- **Published artifacts are written in fluent English** — README, SKILL.md, the arsenal, issues, PRs, releases, commit messages. The audience is the global Rails community. **The runtime audit is the exception: the generated report and step announcements match the language the user writes in.**
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
- **The arsenal is plain markdown** (`arsenal/code-quality.md`, `arsenal/authorization.md`, `arsenal/authentication.md`, …) — the folder is the arsenal, each file is one weapon. The community contributes checks via text-only PRs; the maintainer owns the engine (the RuboCop/cops model).
- **Every bullet that names a concrete, code-level problem carries a tight `# Problem` / `# Fix` example** — a fenced code block right under the bullet, usually two lines (a problem line and a fix line with trailing comments). The example is the point: it turns a description into something recognizable on sight, which is exactly what a developer can't get by asking an agent to "review my code". Keep each minimal so the file still scans — this is reinforced everywhere, but never at the cost of readability. Exempt: methodology/process steps, severity prose, and pure principles that have no single code form.
- **The canonical source for the security arsenal is the official Rails Security Guide** (https://guides.rubyonrails.org/security.html) — distill from it, link findings to its sections, and re-check the arsenal against it on new Rails releases. Prefer it over community checklists, which go stale. (Exception: AI/LLM risks aren't in the Rails guide — `arsenal/ai.md` distills from the OWASP LLM Top 10 and the OWASP LLM Prompt Injection Prevention Cheat Sheet.)
- **Coverage is tracked against the OWASP Top 10 2025**, with OWASP RailsGoat (and its Rails 8 wiki) as the worked-example corpus and regression target. Current mapping: A01→authorization, A02→hardening, A03→cve, A04→cryptography, A05→Brakeman, A06→code-quality+architecture+activerecord (architecture distils archspec's boundary/dependency-direction techniques; activerecord owns ActiveRecord correctness/integrity — callback/transaction timing, query determinism, association integrity), A07→authentication, A08→Brakeman (mass assignment/deserialization)+ci-cd (CI/CD pipeline integrity — `pull_request_target`, script injection, unpinned actions, OIDC vs stored keys), A09→logging, A10→hardening+code-quality. A category with no strong owner is the next weapon to add. **Availability/operational findings fall outside the OWASP web Top 10 and are owned by `arsenal/migrations.md`** (zero-downtime migration safety — table-locking statements, in-migration backfills, expand/contract deploy hazards) **and `arsenal/jobs.md`** (background-job safety — idempotency on retry, secrets/PII in job arguments, stale records as arguments; the secret-in-args case also touches A09 disclosure); both sit in the arsenal deliberately off the OWASP map. **AI/LLM integration is tracked against the OWASP LLM Top 10 2025** (LLM01 prompt injection, LLM02 sensitive-info disclosure, LLM05 improper output handling, LLM06 excessive agency, LLM10 unbounded consumption), cutting across web A03/A06/A09; owned by `arsenal/ai.md`.

## Internal docs

`HANDOFF.md` (gitignored, in Portuguese) holds internal project context for the maintainer. Never quote or copy from it into public artifacts.
