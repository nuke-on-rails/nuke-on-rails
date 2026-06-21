---
name: nuke-on-rails
description: Full health and security audit for Rails apps, the review a principal engineer would do. Runs rubycritic, Brakeman, bundler-audit and ruby_audit, triages every finding with the LLM as judge, brings an OWASP Top 10 arsenal of checks for what scanners miss, and returns one impact-ranked action plan. Use for a Rails project audit, a security and health check, "nuke on rails", or a deep review of a vibecoded Rails app.
disable-model-invocation: true
---

# Nuke on Rails

A full project health audit for Rails apps: *what to refactor, what's vulnerable, and in what order to attack it*. Three deterministic engines do the scanning; you are the judge. The output is **one single list, prioritized by impact**, never tool sections stapled together.

**Respond in the user's language.** Write the step announcements and the *entire* final report in whatever language the user writes in — and that means **every label too**: severity words (`Critical`/`High`/`Medium`/`Low`), section titles (`SCOREBOARD`, `FIX NOW`, `FIX NEXT`, `BIGGEST STRUCTURAL MULTIPLIER`, `RULED OUT`, `NOTES`), and finding fields (`Problem`/`Solution`/`Files`/`Technical details`). The English in these instructions and in `templates/_OUTPUT.md` is the authoring language only — **never emit an English label into a non-English report.** Only fixed tokens stay as-is: the brand `NUKE ON RAILS`, emoji, file paths, code identifiers, CVE ids.

This audit is a recon sweep — and it takes minutes, so announce each step as a dry field-radio line and the user watches the campaign advance. The conceit: you are the recon team scouting the codebase's *own* walls for the breaches a real attacker would use; the enemy is the debt and the holes, never the user. The lines below are authored in English — translate them to the user's language, one short line per step:

- `☢️ NUKE ON RAILS — operation underway`
- `🔭 Recon — scouting the terrain (Rails? Ruby? git history?)…`
- `🎖️ Mobilizing — arming & firing the engines…`
- `🎯 Targets — marking where to concentrate fire (churn × complexity)…`
- `⚔️ Probing the defenses — triaging N alerts, bringing the arsenal…`
- `🧱 Inspecting the walls — reviewing the most battered files…`
- `📋 Briefing the command — consolidating the field report…`

The visible tool calls fill in the rest. The campaign voice lives in these announcements (and the TL;DR framing) **only** — it must never bleed into the findings, which stay sober and literal.

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

**Churn needs git history — verify it before trusting the quadrant.** rubycritic derives churn from `git log`, so a shallow clone (`--depth 1`) or a freshly-initialized repo gives every file the same churn (1 or 0) and the quadrant silently collapses to noise — common exactly in the unfamiliar-repo case this skill targets. If churn is uniform across modules, say so and fall back to ranking by complexity and smell density (rubycritic wraps Reek/Flay/Flog — use the per-module `smells` and `complexity`, not just the score). When possible, unshallow first (`git fetch --unshallow`).

## Step 3 — Triage security findings (you are the judge, not the scanner)

For every Brakeman warning and every bundler-audit CVE:

1. **Read the actual code path.** Confirm the finding is reachable in this app, or kill it as a false positive.
2. **Explain the exploit path** for confirmed findings: who can trigger it, from where, with what impact.
3. **Adversarial verification before it enters the report.** Security findings are held to a higher bar than quality findings: a weak quality finding gets ignored; a false security claim burns trust. If you cannot articulate the exploit path, downgrade it to "theoretical" — never present it as confirmed.

Treat Brakeman's confidence level (`High`/`Medium`/`Weak`) as a prior, not a verdict: a `Weak` warning you confirm by reading the code outranks a `High` warning on an unreachable path. And if `config/brakeman.ignore` exists, re-triage every silenced warning — in an unreviewed codebase, an ignore file often means "made CI pass", not "verified safe".

For dependency and Ruby-version CVEs, apply `arsenal/cve.md` to the bundler-audit and ruby_audit output — it covers deduping, severity priors, reachability triage, insecure gem sources, and the network cross-checks (day-zero and second-opinion) that close the advisory database's lag and coverage gaps.

Then bring the security arsenal to the routes file, the sensitive controllers and models, and the production config:

- `arsenal/authorization.md` — IDOR, missing authorization, attack surface
- `arsenal/authentication.md` — auth stack, Devise config, custom strategies, sessions
- `arsenal/secrets.md` — committed keys and hardcoded credentials
- `arsenal/hardening.md` — TLS, CSP, CSRF config, mounted dashboards, uploads
- `arsenal/api.md` — JSON over-exposure, CORS, rate limiting, webhooks (skip if the app has no JSON endpoints)
- `arsenal/cryptography.md` — encryption oracles, hand-rolled crypto, weak hashing, plaintext sensitive columns
- `arsenal/logging.md` — sensitive data in logs, missing audit trail on security-critical events
- `arsenal/ai.md` — prompt injection, LLM output rendered as XSS, PII leaked to model APIs, over-powered tool-use (skip if the app makes no LLM calls)
- `arsenal/ci-cd.md` — pipeline security in `.github/workflows/` / `.gitlab-ci.yml` / `Jenkinsfile`: `pull_request_target` running fork code, untrusted `${{ }}` in shells, unpinned actions, long-lived cloud keys (skip if the repo has no CI config)

They cover what Brakeman can't reach. The arsenal *covers* those areas; it does not *guarantee* them. Be explicit about that distinction in the report.

## Step 4 — Quality review of the hotspots

Apply `arsenal/code-quality.md` — the thermo-nuclear standards, translated to the Rails idiom — to the hotspot files from Step 2. That check is the default quality bar for this skill: ambitious structural findings, not cosmetic nits.

Then apply `arsenal/migrations.md` to the recent migrations in `db/migrate/` (read against `db/schema.rb` for table sizes) — the availability weapon no engine owns. A migration that locks or rewrites a large, traffic-heavy table, or drops/renames a column ahead of the code that uses it, is a scheduled outage, not a code smell: rank it accordingly.

Also apply `arsenal/architecture.md` to the `app/` layer folders and the hotspots — dependency *direction* and cycles, which rubycritic's per-file scores miss: a model reaching up into the web layer, two namespaces that reference each other, or Zeitwerk name/path drift.

And apply `arsenal/activerecord.md` to `app/models/` and the query-heavy hotspots — the correctness/integrity of the ActiveRecord calls themselves (a side effect in `after_save` racing the transaction, `where(...).first` with no order, `has_many` without `dependent:`), which the structural weapons don't judge.

And apply `arsenal/jobs.md` to `app/jobs/` and the enqueue sites — background-job safety the engines can't model: a non-idempotent job that repeats its side effect on retry (double-charge), secrets/PII in job arguments (persisted in the queue store and shown on the dashboard), and records passed instead of ids.

## Step 5 — The report (output)

This section governs *what goes in* the report and *how it's judged* — language, tone, ranking, and the closing summary. **The visual structure it's rendered in — terminal-native plaintext, no markdown — lives in `templates/_OUTPUT.md`; the report must match that skeleton.** Write the whole report in the user's language (see the top of this file).

**Open with a short status banner** — just write it, don't print a "Status banner" label. A few lines of orientation: project type and Rails/Ruby versions (note any substitution, e.g. running under a different Ruby), whether git history made the churn quadrant reliable, the engines that ran with headline counts, and one honest line on coverage (weapons cover, they don't guarantee). Open the TL;DR as a dry field report to the command, in keeping with the recon-campaign voice of the step announcements (English example, for tone only: "Field report: the position isn't on fire — but the back gate is unlocked, and that's where they get in."). **Write it natively in the user's language** so it actually lands; never translate the example literally. The campaign voice lives in the banner/TL;DR framing only; every finding stays sober and precise.

Right after the banner, open the body with the **severity scoreboard** — the triaged counts on one inline line, count before the label (never a markdown table; see `templates/_OUTPUT.md`):

```text
🔴 1 Critical   🟠 4 High   🟡 6 Medium   🟢 5 Low
```

Dependency-risk totals and any end-of-life flag stay in the opening banner — don't repeat them here.

Then the findings, as one list ranked by impact:

1. **Confirmed exploitable security findings** (an IDOR in a payments controller outranks everything).
2. **CVEs** actually reachable from the app's usage.
3. **High-churn × high-complexity quality hotspots** (a fat model that changes weekly outranks a theoretical warning).
4. **Theoretical security warnings** that survived triage but lack a demonstrated exploit path.
5. **Remaining quality findings**, worst first.

**Keep each finding tight, scannable, and in plain human language.** Problem and Solution read for a non-expert stakeholder *and* an exhausted senior — everyday words, no method names or line refs. Lead with a one-line headline: severity + what it is, in plain language (no finding number, no effort estimate). Then the plain-language problem and the concrete fix. Push the technical specifics — method names, line refs, the reachable exploit path, and any churn/complexity metrics — into an optional `Technical details` field (see `templates/_OUTPUT.md`); a finding with nothing concrete to prove omits it. Show code only when a few lines make the point faster than words. No multi-paragraph exploit essays: a senior should grasp each finding in seconds and know the next move. When an issue recurs, cite the 2–5 strongest `file:line` locations and note "+ ~N similar", not every instance. Write the way Rails reads — friendly, direct, no ceremony. Don't print scaffolding labels ("Status banner", "Findings"); let the report flow.

**Close with the plan** so the reader leaves with a next move (the scoreboard already opened the body, up top):

1. **Fix now** — ranked by **leverage**: the most risk or debt removed for the least *change*, not impact alone. A confirmed critical *and* a high-impact quick win (a version bump, a one-line config) both belong here; a broad, high-risk structural change drops to *Fix next* even when impactful. Don't estimate time — how long a fix takes depends on the team and their tooling; judge by scope and blast radius, and describe that in words. Tiebreakers: anything that unblocks other work (a verification baseline, characterization tests) floats up; a high-confidence security finding floats above an equivalent-leverage non-security one; prefer fixes with a clean way to verify them. Each a tight, parallel line — subject then the move — the "if you do nothing else" list.
2. **Fix next** — the remaining high/medium, grouped tersely.
3. **Biggest structural multiplier** — one line naming the single refactor that removes the most risk or debt at once.

If the scanners raised things you ruled out, say so — but as **one compact line near the end** (name them, a one-word reason each: false positive / not reachable / dev-only / wrong target), never a verbose section. The reader should know the noise was checked, not missed.

**The report is sensitive.** It enumerates live, confirmed vulnerabilities and their exploit paths — it belongs with the people who can fix the app, not a public issue tracker or an open channel. If findings are published anywhere shared or public, redact the security specifics (exploit path, credential location) first.

The whole thing reads like a plan a principal engineer would hand you, not a tool dump.
