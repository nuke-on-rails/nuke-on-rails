# Lens: Dependency & Runtime CVEs

bundler-audit and ruby_audit produce the raw list; this lens turns it into triaged findings. The engines compare versions against ruby-advisory-db — they cannot tell whether a vulnerability is *reachable* in this app, and they cannot see advisories the database hasn't absorbed. Both jobs land here.

Honesty first: an empty engine result means "no known advisories in ruby-advisory-db", never "dependencies are secure". Phrase the report accordingly.

## Normalize the engine output

- **Dedupe by `(gem name, advisory id)`** — a multi-platform `Gemfile.lock` repeats the same advisory once per platform (a single nokogiri advisory can appear 7×).
- **Severity prior:** `cvss_v3`; it can be null — fall back to `criticality`, then to reading the description.
- **Concrete fix:** `patched_versions`, as "bump gem X to ≥ Y".
- **Insecure gem sources** (`git://`, `http://` in the Gemfile) come in the same output: triage who controls that source and whether it's pinned to a commit — an unpinned gem from a personal fork is a supply-chain finding, not a style note.

## Reachability triage

Installed is not exploitable. For each advisory, ask whether the app actually exercises the vulnerable component:

- A net-imap CVE in an app that never touches IMAP → **theoretical** (recommend the bump anyway — it's cheap insurance).
- Rack, puma, erb and friends are reachable *by construction* in any web-facing Rails app — don't theoretical-ize the request path.
- Confirmed reachable + high CVSS competes for the top of the report; theoretical ranks below quality hotspots but stays in the list with its one-line fix.

## Network cross-checks (the database's blind spots)

The agent has web access; the engines don't. Two blind spots, three checks — all of them scrape/network-based: if a page changes shape or you're offline, skip and say so in the report.

- **Day-zero, Ruby core and Rails:** fetch https://www.ruby-lang.org/en/security/ and the latest Rails security announcements; compare recent advisories against the app's Ruby and Rails versions. The advisory db lags announcements by hours-to-days; anything inside that window appears in no engine's output. Report as a day-zero finding with the official link.
- **Day-zero, gems:** fetch https://security.snyk.io/vuln/rubygems once — server-rendered, ~30 newest RubyGems disclosures, gem name in each slug (`SNYK-RUBY-<GEM>-…`). Intersect the names with `Gemfile.lock`; triage only the matches.
- **Second opinion, critical gems:** for the security-critical gems only (web server, auth, file/XML parsing — not the whole lockfile), `POST https://api.osv.dev/v1/querybatch` with `{"package": {"name": …, "ecosystem": "RubyGems"}, "version": …}`. OSV aggregates the GitHub Advisory Database and covers what ruby-advisory-db misses. The batch response is an *index* — ids and timestamps only. Match those ids (and their CVE aliases) against what the engines already reported, and fetch details (`GET /v1/vulns/{id}`) **only for the ids the engines missed** — those are the coverage-gap findings. In the detail record: `aliases` maps GHSA↔CVE, `database_specific.severity` has the severity label (the `severity` field is a CVSS vector string, not a number), and `affected.ranges.events.fixed` is the bump target. Triage like any other CVE.

## Reporting

Group by gem, not by advisory: "bump rack 3.2.5 → ≥ 3.2.6 — clears 13 advisories (worst: CVSS 7.5 request smuggling)" is one actionable item, not thirteen line items. For each confirmed finding, one blast-radius sentence: what an attacker gets. End the CVE section with the single command that clears the most findings at once.
