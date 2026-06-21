# Weapon: <Weapon Name>

<One or two sentences framing the territory the scanners miss. What does rubycritic/Brakeman/bundler-audit NOT see here, and why does reading the code with the app's domain in mind catch it? Name the files to read (`app/...`, `config/...`).>

<Optional: `Reference: <official source>` — for security weapons, link the Rails Security Guide section. Distil from it; don't copy.>

## <First problem topic>

<One or two sentences: what the problem is, the detectable signal, and why it matters here. Then a tight Problem/Fix example — usually two lines, the labels as trailing comments. The example is required for any code-level problem; keep it minimal so the file still scans.>

```ruby
Model.find(params[:id])                # Problem — what's wrong, in a few words
current_user.things.find(params[:id])  # Fix — the one-line correction
```

## <Second problem topic>

<Same shape. One topic = one coherent finding, with its own example.>

```ruby
bad_thing   # Problem — ...
good_thing  # Fix — ...
```

<Repeat for each distinct finding. Hold an opinion; acknowledge a counter-position in a clause, not an essay. Say "this weapon covers X; it does not guarantee it" where that honesty matters. Cross-reference sibling weapons with `arsenal/<other>.md` instead of duplicating.>

## Severity and remedies

<Rank the findings by impact — what actually costs the most (a confirmed exploit on money/PII > a theoretical warning > a quality nit). For security findings, state the exploit in one sentence and hold them to the higher bar: no articulable exploit path → "theoretical", never "confirmed". Name the concrete remedy (often a one-liner, or the canonical gem: strong_migrations, rack-attack, discard, …). Cross-reference related weapons.>

<!--
  Checklist before opening the PR (delete this comment):
  - Every code-level problem bullet has a # Problem / # Fix example.
  - Catches what the engines don't (no Brakeman/rubycritic duplication).
  - Security findings describe a detectable signal AND an exploit path.
  - Wired in: SKILL.md (Step 3 security / Step 4 quality), README <details> block, CLAUDE.md OWASP map.
  - Fluent English; opinionated, not "it depends".
-->
