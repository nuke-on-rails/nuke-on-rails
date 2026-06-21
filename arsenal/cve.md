# Lens: Dependency & Runtime CVEs

bundler-audit and ruby_audit produce the raw list; this lens turns it into triaged findings. The engines compare versions against ruby-advisory-db — they cannot tell whether a vulnerability is *reachable* in this app, and they cannot see advisories the database hasn't absorbed. Both jobs land here.

Honesty first: an empty engine result means "no known advisories in ruby-advisory-db", never "dependencies are secure". Phrase the report accordingly.

## Normalize the engine output

- **Dedupe by `(gem name, advisory id)`** — a multi-platform `Gemfile.lock` repeats the same advisory once per platform (a single nokogiri advisory can appear 7×).
- **Severity prior:** `cvss_v3`; it can be null — fall back to `criticality`, then to reading the description.
- **Concrete fix:** `patched_versions`, as "bump gem X to ≥ Y".
- **Insecure gem sources** (`git://`, `http://` in the Gemfile) come in the same output: triage who controls that source and whether it's pinned to a commit — an unpinned gem from a personal fork is a supply-chain finding, not a style note.

```ruby
# Problem — a gem pulled over an unencrypted protocol from a personal fork, unpinned
gem "acme", git: "git://github.com/some-user/acme.git"   # git:// is unauthenticated; HEAD floats

# Fix — HTTPS source pinned to a reviewed commit (or the canonical RubyGems release)
gem "acme", git: "https://github.com/some-user/acme.git", ref: "a1b2c3d"
```

## JavaScript dependencies (the other half of the lockfile)

A modern Rails app ships JS the gem engines never see. Detect the toolchain and audit it: `bin/importmap audit` (importmap-rails), or `yarn npm audit` / `npm audit` when a `package.json`/`yarn.lock` is present. Triage the hits exactly like gem CVEs — reachability, severity, fix version. If the app has no JS dependencies, note that the check found none rather than skipping silently.

Beyond known CVEs, importmap carries a **supply-chain** exposure: a `pin` that points at an external CDN URL loads runtime JS from a third party with no integrity guarantee — a compromised or hijacked CDN runs arbitrary code in every user's browser (OWASP A08). Vendor the pins so the app serves them.

```ruby
pin "react", to: "https://ga.jspm.io/npm:react@18/index.js"  # Problem — runtime JS from a 3rd party, no integrity
# Fix — vendor it: `bin/importmap pin react` downloads to vendor/javascript/ and serves it from your app
```

## Reachability triage

Installed is not exploitable. For each advisory, ask whether the app actually exercises the vulnerable component:

- A net-imap CVE in an app that never touches IMAP → **theoretical** (recommend the bump anyway — it's cheap insurance).
- Rack, puma, erb and friends are reachable *by construction* in any web-facing Rails app — don't theoretical-ize the request path.
- Confirmed reachable + high CVSS competes for the top of the report; theoretical ranks below quality hotspots but stays in the list with its one-line fix.

```text
# Problem — a reachable advisory in a web-facing gem (bundler-audit output)
Name: rack   Version: 3.1.7
Advisory: CVE-2025-XXXXX  (CVSS 7.5 — HTTP request smuggling)
Solution: upgrade to >= 3.1.8
→ rack is in every request path by construction: reachable, not "theoretical".

# Fix
bundle update rack --conservative   # clears the advisory with the smallest possible bump
```

## Network cross-checks (the database's blind spots)

The agent has web access; the engines don't. Two blind spots, three checks — all of them scrape/network-based: if a page changes shape or you're offline, skip and say so in the report.

- **Day-zero, Ruby core and Rails:** fetch https://www.ruby-lang.org/en/security/ and the latest Rails security announcements; compare recent advisories against the app's Ruby and Rails versions. The advisory db lags announcements by hours-to-days; anything inside that window appears in no engine's output. Report as a day-zero finding with the official link.
- **Day-zero, gems:** fetch https://security.snyk.io/vuln/rubygems once — server-rendered, ~30 newest RubyGems disclosures, gem name in each slug (`SNYK-RUBY-<GEM>-…`). Intersect the names with `Gemfile.lock`; triage only the matches.
- **Second opinion, critical gems:** for the security-critical gems only (web server, auth, file/XML parsing — not the whole lockfile), `POST https://api.osv.dev/v1/querybatch` with `{"package": {"name": …, "ecosystem": "RubyGems"}, "version": …}`. OSV aggregates the GitHub Advisory Database and covers what ruby-advisory-db misses. The batch response is an *index* — ids and timestamps only. Match those ids (and their CVE aliases) against what the engines already reported, and fetch details (`GET /v1/vulns/{id}`) **only for the ids the engines missed** — those are the coverage-gap findings. In the detail record: `aliases` maps GHSA↔CVE, `database_specific.severity` has the severity label (the `severity` field is a CVSS vector string, not a number), and `affected.ranges.events.fixed` is the bump target. Triage like any other CVE.

## End-of-life runtime and framework

The engines flag *known CVEs against the current version*; they do not flag that a version is **past end-of-life and will never receive another patch**. An EOL Rails or Ruby with a clean bundler-audit today is still a standing critical risk — the next CVE (like the March 2026 Rails path-traversal/XSS/DoS fixes) simply won't be backported. Auditors treat EOL as a critical compliance finding (PCI DSS, HIPAA).

Check both clocks independently (the "dual EOL" problem) — Ruby from `.ruby-version`/`Gemfile`, Rails from `Gemfile.lock`. EOL dates move, so verify against current support status (endoflife.date, the Rails maintenance policy) when online rather than trusting a hardcoded table. Snapshot as of 2026 for reference: **Ruby** ≤3.1 EOL, 3.2 EOL March 2026, 3.3–3.4 supported; **Rails** ≤7.1 EOL, 7.2 security-only until Aug 2026, 8.0 supported. Report an EOL runtime/framework as confirmed-critical with the concrete remedy: the nearest supported upgrade target.

```text
# Problem — the clock the engines DON'T flag: a runtime past end-of-life
ruby 3.1.x   →   EOL: no future security patch will ever be backported
# bundler-audit can be clean today and this is still a standing critical risk —
# the next Ruby/Rails CVE simply won't reach this version.

# Fix — plan the upgrade to a supported line (Ruby 3.3+, a maintained Rails)
```

## Reporting

Group by gem, not by advisory: "bump rack 3.2.5 → ≥ 3.2.6 — clears 13 advisories (worst: CVSS 7.5 request smuggling)" is one actionable item, not thirteen line items. For each confirmed finding, one blast-radius sentence: what an attacker gets. End the CVE section with the single command that clears the most findings at once.
