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

bundler-audit also flags insecure gem sources (`git://`, `http://`) — triage those too.

If an engine fails, degrade gracefully: report which engine was skipped and why, and continue with the others.

## Step 2 — Decide where to spend context (churn × complexity)

Use rubycritic's churn × complexity data to pick the hotspots: files that are both complex **and** change often. Those are the files you read deeply. Never review the codebase uniformly — a complex file nobody touches is a lower priority than a moderately complex file that changes every week.

## Step 3 — Triage security findings (you are the judge, not the scanner)

For every Brakeman warning and every bundler-audit CVE:

1. **Read the actual code path.** Confirm the finding is reachable in this app, or kill it as a false positive.
2. **Explain the exploit path** for confirmed findings: who can trigger it, from where, with what impact.
3. **Adversarial verification before it enters the report.** Security findings are held to a higher bar than quality findings: a weak quality finding gets ignored; a false security claim burns trust. If you cannot articulate the exploit path, downgrade it to "theoretical" — never present it as confirmed.

Treat Brakeman's confidence level (`High`/`Medium`/`Weak`) as a prior, not a verdict: a `Weak` warning you confirm by reading the code outranks a `High` warning on an unreachable path. And if `config/brakeman.ignore` exists, re-triage every silenced warning — in an unreviewed codebase, an ignore file often means "made CI pass", not "verified safe".

For CVEs, first dedupe results by `(gem name, advisory id)` — a multi-platform `Gemfile.lock` repeats the same advisory once per platform. Use the advisory's `cvss_v3` as the severity prior (it can be null — fall back to `criticality`, then to reading the description) and `patched_versions` as the concrete fix ("bump gem X to ≥ Y"). And be honest about coverage: bundler-audit and ruby_audit check against ruby-advisory-db, a community database that is not exhaustive — an empty result means "no known advisories", never "dependencies are secure". Phrase the report accordingly.

**Day-zero cross-check (network permitting).** ruby-advisory-db lags the official announcements by hours or days — an audit run inside that window misses freshly published CVEs. You are an agent with web access; the gem-based engines are not. Fetch https://www.ruby-lang.org/en/security/ and the latest Rails security announcements, and compare the recent advisories against the app's Ruby and Rails versions. For gem-level day-zero coverage, fetch https://security.snyk.io/vuln/rubygems once — the listing is server-rendered and carries the ~30 newest RubyGems disclosures with the gem name in each slug (`SNYK-RUBY-<GEM>-…`); intersect those names with the `Gemfile.lock` and triage only the matches. Anything announced but not yet absorbed by ruby-advisory-db appears in no engine's output — report it as a day-zero finding with a link to the advisory. These are scrape-based checks: if a page changes shape or you are offline, skip and say so in the report.

**Second-opinion cross-check (network permitting).** For the app's security-critical gems (web server, auth, file/XML parsing — not the whole lockfile), query OSV.dev's free batch API, which aggregates the GitHub Advisory Database and covers what ruby-advisory-db misses: `POST https://api.osv.dev/v1/querybatch` with each gem's name (`"ecosystem": "RubyGems"`) and locked version. A hit that bundler-audit didn't report is a coverage-gap finding — triage it like any other CVE.

Then apply the security lenses — `lenses/authorization.md` (IDOR, missing authorization, attack surface), `lenses/authentication.md` (auth stack, Devise config, custom strategies, sessions) and `lenses/secrets.md` (committed keys and hardcoded credentials) — to the routes file and the sensitive controllers, models and initializers. They cover what Brakeman can't reach. Lenses *cover* those areas; they do not *guarantee* them. Be explicit about that distinction in the report.

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
