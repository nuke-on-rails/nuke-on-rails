---
name: nuke-on-rails
description: Full health audit for Rails apps — the review a principal engineer would do. Runs rubycritic, Brakeman and bundler-audit, triages every finding with the LLM as judge, and produces one impact-ranked action plan. Use for a Rails project audit, a security and health check, "nuke on rails", or a deep review of a vibecoded Rails app.
disable-model-invocation: true
---

# Nuke on Rails

A full project health audit for Rails apps: *what to refactor, what's vulnerable, and in what order to attack it*. Three deterministic engines do the scanning; you are the judge. The output is **one single list, prioritized by impact** — never tool sections stapled together.

## Step 0 — Detect the project

- **Rails app** (`config/application.rb` exists, or the `rails` gem is in the Gemfile): run the full audit.
- **Plain Ruby** (Gemfile or gemspec, no Rails): graceful degradation — rubycritic and bundler-audit run, Brakeman is skipped. Say so explicitly in the report header.
- **Neither**: stop and tell the user this skill audits Ruby/Rails projects.

## Step 1 — Install and run the engines

Zero-dependency principle: the skill brings its own tools. Never touch the app's Gemfile.

```sh
gem list rubycritic -i || gem install rubycritic
gem list brakeman -i || gem install brakeman          # Rails only
gem list bundler-audit -i || gem install bundler-audit
```

Then run all three and capture machine-readable output:

```sh
rubycritic --format json --no-browser <app paths>
brakeman --format json --quiet                         # Rails only
bundle-audit check --update
```

If an engine fails, degrade gracefully: report which engine was skipped and why, and continue with the others.

## Step 2 — Decide where to spend context (churn × complexity)

Use rubycritic's churn × complexity data to pick the hotspots: files that are both complex **and** change often. Those are the files you read deeply. Never review the codebase uniformly — a complex file nobody touches is a lower priority than a moderately complex file that changes every week.

## Step 3 — Triage security findings (you are the judge, not the scanner)

For every Brakeman warning and every bundler-audit CVE:

1. **Read the actual code path.** Confirm the finding is reachable in this app, or kill it as a false positive.
2. **Explain the exploit path** for confirmed findings: who can trigger it, from where, with what impact.
3. **Adversarial verification before it enters the report.** Security findings are held to a higher bar than quality findings: a weak quality finding gets ignored; a false security claim burns trust. If you cannot articulate the exploit path, downgrade it to "theoretical" — never present it as confirmed.

Then apply the security lenses in `lenses/` to the hotspot controllers and models — they cover what Brakeman can't reach (IDOR, missing authorization, business-logic flaws). Lenses *cover* those areas; they do not *guarantee* them. Be explicit about that distinction in the report.

## Step 4 — Quality review of the hotspots

Apply `lenses/code-quality.md` — the thermo-nuclear standards, translated to the Rails idiom — to the hotspot files from Step 2. That lens is the default quality bar for this skill: ambitious structural findings, not cosmetic nits.

## Step 5 — One impact-ranked report

Merge everything into a single ordered list. Ranking heuristic:

1. **Confirmed exploitable security findings** (an IDOR in a payments controller outranks everything).
2. **CVEs in gems** that are actually reachable from the app's usage.
3. **High-churn × high-complexity quality hotspots** (a fat model that changes weekly outranks a theoretical warning).
4. **Theoretical security warnings** that survived triage but lack a demonstrated exploit path.
5. **Remaining quality findings**, worst first.

For each item: severity, file/line, why it matters in *this* app, and the concrete first step to fix it. End with a short "if you only fix three things" section.

The report must read like a plan a principal engineer would hand you — not a tool dump.
