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

**Ruby version trap.** A `.ruby-version` (or `Gemfile`-pinned Ruby) often names a version not installed on this machine — common in unfamiliar or AI-generated repos, and it makes rubycritic and brakeman abort before producing output. The engines do static analysis and don't need the app's exact Ruby. If the pinned version is missing, run them under an available Ruby (e.g. set a temporary `.ruby-version`, or invoke via `rbenv local <installed>` / a system Ruby) rather than failing — and note the substitution in the report. Never install the pinned Ruby just to satisfy the lock.

## Step 1 — Install and run the engines

Zero-dependency principle: the skill brings its own tools. Never touch the app's Gemfile.

```sh
gem list rubycritic -i || gem install rubycritic
gem list brakeman -i || gem install brakeman          # Rails only
gem list bundler-audit -i || gem install bundler-audit
gem list ruby_audit -i || gem install ruby_audit
```

Then run them all and capture machine-readable output (commands validated on a real Rails 8 app):

```sh
OUT=$(mktemp -d)
rubycritic app --format json --no-browser --path "$OUT/rubycritic"   # --path keeps output files out of the audited repo
brakeman --format json --quiet -o "$OUT/brakeman.json"               # Rails only
bundle-audit update                                                  # update the db in a separate step:
bundle-audit check --format json > "$OUT/bundler-audit.json"         # --update mixed in pollutes the JSON on stdout
(cd "$OUT" && ruby-audit check)                                      # run OUTSIDE the app dir — inside a bundled
                                                                     # project the executable fails to load (Ruby 3.4+);
                                                                     # make sure the app's Ruby version is still active
```

If an engine fails, degrade gracefully: report which engine was skipped and why, and continue with the others.

## Step 2 — Decide where to spend context (churn × complexity)

Use rubycritic's churn × complexity data to pick the hotspots: files that are both complex **and** change often. Those are the files you read deeply. Never review the codebase uniformly — a complex file nobody touches is a lower priority than a moderately complex file that changes every week.

## Step 3 — Triage security findings (you are the judge, not the scanner)

For every Brakeman warning and every bundler-audit CVE:

1. **Read the actual code path.** Confirm the finding is reachable in this app, or kill it as a false positive.
2. **Explain the exploit path** for confirmed findings: who can trigger it, from where, with what impact.
3. **Adversarial verification before it enters the report.** Security findings are held to a higher bar than quality findings: a weak quality finding gets ignored; a false security claim burns trust. If you cannot articulate the exploit path, downgrade it to "theoretical" — never present it as confirmed.

Treat Brakeman's confidence level (`High`/`Medium`/`Weak`) as a prior, not a verdict: a `Weak` warning you confirm by reading the code outranks a `High` warning on an unreachable path. And if `config/brakeman.ignore` exists, re-triage every silenced warning — in an unreviewed codebase, an ignore file often means "made CI pass", not "verified safe".

For dependency and Ruby-version CVEs, apply `lenses/cve.md` to the bundler-audit and ruby_audit output — it covers deduping, severity priors, reachability triage, insecure gem sources, and the network cross-checks (day-zero and second-opinion) that close the advisory database's lag and coverage gaps.

Then apply the security lenses to the routes file, the sensitive controllers and models, and the production config:

- `lenses/authorization.md` — IDOR, missing authorization, attack surface
- `lenses/authentication.md` — auth stack, Devise config, custom strategies, sessions
- `lenses/secrets.md` — committed keys and hardcoded credentials
- `lenses/hardening.md` — TLS, CSP, CSRF config, mounted dashboards, uploads
- `lenses/api.md` — JSON over-exposure, CORS, rate limiting, webhooks (skip if the app has no JSON endpoints)

They cover what Brakeman can't reach. Lenses *cover* those areas; they do not *guarantee* them. Be explicit about that distinction in the report.

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
