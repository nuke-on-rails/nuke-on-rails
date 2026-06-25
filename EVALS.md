# Evals — does the audit still catch what it should?

The quality bar (`CLAUDE.md`) is: *work first time on any Rails repo, zero setup, and produce a result a developer couldn't get by just asking an agent to "review my code".* This file makes that bar **verifiable** and guards against regressions as weapons evolve.

The anchor corpus is **OWASP RailsGoat** — a deliberately-vulnerable Rails app with planted flaws mapped to the OWASP Top 10 (and its Rails 8 update). It's the regression target named in `CLAUDE.md`. The audit is LLM-driven, so these evals are **semi-manual**: run the audit, read the report against the must-catch list below, record any miss. They are not a deterministic CI assertion.

## How to run

```sh
git clone https://github.com/OWASP/railsgoat.git /tmp/railsgoat
cd /tmp/railsgoat
# then run the Nuke on Rails audit on this directory.
```

RailsGoat pins an old Ruby; if it isn't installed, the **Ruby-version trap** guidance in `SKILL.md` Step 0 applies (run the engines under an available Ruby, note the substitution). Pin to a known RailsGoat revision so the must-catch list stays stable; re-verify after bumping it.

## Must-catch corpus (RailsGoat)

Each planted class must appear in the report, ranked roughly in this order. ⭐ marks the findings **the scanners can't reach** — the ones that prove the arsenal earns its place beyond "just ask an agent". A miss on a ⭐ is a release blocker.

| Planted vulnerability | Weapon | OWASP 2025 | Who catches it |
|---|---|---|---|
| ⭐ **IDOR / missing access control** (editing another user's account by changing the id) | `authorization.md` | A01 | LLM judge — Brakeman is blind to it |
| ⭐ **Reused encryption/signing oracle** (`encrypt_value` minting both a trust token and user-readable data) | `cryptography.md` | A04 | LLM judge — the weapon names this as RailsGoat's planted flaw |
| **Mass assignment → privilege escalation** (`admin`/`role` via params) | `authorization.md` + Brakeman | A01/A08 | Brakeman flags some; LLM confirms the semantic case |
| **SQL injection** (string-interpolated `where`/raw SQL) | Brakeman + triage | A03 | Brakeman; LLM triages reachability |
| **Command injection** (`system`/backticks on user input) | Brakeman + triage | A03 | Brakeman |
| **XSS** (reflected and stored, `html_safe`/`raw` on user content) | Brakeman + `hardening.md` | A03 | Brakeman + the html_safe checks |
| **Weak password hashing** (MD5-style digest) | `cryptography.md` | A02 | LLM judge |
| **Sensitive data exposure / hardcoded secrets** | `secrets.md` + `logging.md` | A02/A09 | deterministic greps + LLM |
| **CSRF disabled / weak session config** | `hardening.md` + `authentication.md` | A05/A07 | LLM judge |
| **Unvalidated redirect** (open redirect) | `authentication.md` / `api.md` | A01 | LLM judge |
| **Known-vulnerable gems** (RailsGoat ships old, CVE-laden gems) | `cve.md` | A06 | bundler-audit + reachability triage |

**Ranking check, not just presence:** the report must put the ⭐ confirmed-exploitable findings (IDOR, the encryption oracle) *above* the CVEs and quality findings — that's the "one impact-ranked list" promise. A report that lists everything but ranks a fat-model smell above the IDOR has failed even if it "caught" both.

## Corpus gaps (weapons RailsGoat does not exercise)

RailsGoat predates several of our weapons and doesn't plant these. Validate them with a small crafted fixture (a single file with the planted pattern) or by review — never assume "no RailsGoat finding" means the weapon works:

- `jobs.md` — a non-idempotent `ChargeCustomerJob`; a secret passed as a job argument.
- `migrations.md` — a `db/migrate/` that adds a non-concurrent index / a `NOT NULL` without default; a `TRUNCATE`.
- `ci-cd.md` — a `.github/workflows/*.yml` with `pull_request_target` checking out the PR head; an unpinned third-party action.
- `architecture.md` — a model that references a controller; two namespaces that reference each other.
- `activerecord.md` — a side effect in `after_save`; a `where(...).first` with no order; a `has_many` without `dependent:`.
- `authorization.md` (value-field mass assignment) — RailsGoat plants `admin`/`role` escalation but not the **economic** case: a controller that does `current_user.update(params.permit(:name, :credits))` — ownership correct, token valid, yet `credits` (a server-authority column) is permitted. The report must flag it; a skill that only catches `role`/`admin` misses the more common SaaS exploit (free credits / plan upgrade).
- `ai.md` — a chat endpoint that interpolates `params` into a prompt and renders the response with `raw`.

These fixtures live in the reviewer's scratch directory, not the repo (the skill is zero-dependency and ships no app code).

## What "pass" means

1. Every ⭐ finding is in the report, **confirmed** (not "theoretical"), with an articulated exploit path.
2. The impact ranking holds: confirmed exploits above CVEs above quality findings.
3. No false security claim — every "confirmed" survives the report's own adversarial bar.
4. Degradation is honest: if an engine was skipped (e.g. Brakeman on plain Ruby, or a Ruby-version substitution), the report says so.

A miss on rule 1 or a violation of rule 3 blocks a release. Rules 2 and 4 are quality regressions worth a fix before shipping.

## Note

This is a manual/semi-manual harness on purpose: the audit's value is the LLM's judgment, which a deterministic test can't assert. Re-run it before cutting a release, and whenever a weapon changes in a way that could affect a must-catch finding.
