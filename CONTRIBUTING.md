# Contributing to Nuke on Rails

Thanks for wanting to sharpen the arsenal. This guide is short and opinionated, like the audit itself.

The model is RuboCop/cops: **the maintainer owns the engine (`SKILL.md`, the pipeline); the community grows the arsenal** — the plain-markdown checks in `arsenal/` that the audit applies on top of the scanners. A new check is a text-only PR. You do not need to write Ruby to contribute a weapon.

## Before you start

1. Read `CLAUDE.md` — it's the project context and the quality bar.
2. Check the **OWASP Top 10 2025 coverage map** in `CLAUDE.md`. A category with no strong owner is the next weapon worth adding; an existing weapon usually wants *deepening*, not a second file. Open an issue before writing a whole new weapon, so we can agree on scope and where it lives.
3. The bar is high: the audit must produce a result a developer *couldn't get by just asking an agent to "review my code"*. A weapon earns its place by catching what the three engines (rubycritic, Brakeman, bundler-audit) can't reach — the semantic, the configural, the cross-file.

## What a weapon is

One file in `arsenal/` is one weapon (`arsenal/authorization.md`, `arsenal/jobs.md`, …). Each reads as a **lens** a principal engineer applies: a short framing of the territory the scanners miss, then the concrete findings, then how to rank them. Copy `arsenal/_TEMPLATE.md` to start — it encodes the house structure.

### Non-negotiables for any check

- **Every bullet that names a concrete, code-level problem carries a tight `# Problem` / `# Fix` example** — a fenced block right under the bullet, usually two lines. The example is the point; it turns a description into something recognizable on sight. Keep it minimal so the file still scans. (Methodology/severity/pure-principle bullets are exempt.)
- **It catches what the engines don't.** If Brakeman or rubycritic already flags it deterministically, don't re-document it — add the *semantic* layer they miss, or skip it.
- **Hold an opinion.** State the finding and the fix directly; acknowledge a counter-position in a clause, not an "it depends" essay. The voice is the Rails guides crossed with a code review: friendly, direct, no ceremony.
- **"Covers, not guarantees."** A lens *covers* an area; it doesn't *prove* the absence of a bug. Say so where it matters.

### Security findings are held to a higher bar

A weak quality finding gets ignored; a false security claim in a report burns trust. Any security check must describe a **detectable signal** and an **articulable exploit path**. If you can't state who exploits it and how, it's "theoretical", not "confirmed" — write it that way.

## How to add or change a weapon

```bash
# New weapon:
cp arsenal/_TEMPLATE.md arsenal/your-weapon.md
# then wire it in:
#   1. SKILL.md  — add it to Step 3 (security) or Step 4 (quality/operational)
#   2. README.md — add a <details> block to the arsenal list
#   3. CLAUDE.md  — add it to the OWASP coverage map (or note why it's off-map)
# Deepening an existing weapon: just edit its file (+ README bullet if the summary changes).
```

Then open a PR. Keep the diff focused — one weapon or one cohesive set of checks per PR, so it's reviewable.

## What we reject

- **Engine code.** The arsenal is markdown. A detection *script the agent runs and reads* (like the `git ls-files` greps in `arsenal/secrets.md`) is fine; a gem or a library to `require` is not.
- **Checks without a `# Problem` / `# Fix` example.**
- **Duplicates of what Brakeman / rubycritic / bundler-audit already catch**, with no added semantic layer.
- **"It depends" essays.** We hold opinions and explain them.
- **Cosmetic nits** (`Time.now` vs `Time.current`, style) — RuboCop owns those. We flag structural and security issues, not lint.
- **Copied prose.** Distill the *technique* into your own words and examples (ideas aren't copyrightable; expression is). Cite the source. Prefer the [official Rails Security Guide](https://guides.rubyonrails.org/security.html) over community checklists — they go stale.

## What we'd love

- A weapon for an OWASP category with no strong owner (see the map in `CLAUDE.md`).
- Deepening an existing weapon: a new finding with a real, sanitized before/after from your own codebase.
- A regression example from OWASP RailsGoat that a current weapon should have caught and didn't.

## Hard rules (from `CLAUDE.md`)

- **Published artifacts are written in fluent English** — the arsenal, README, PRs, commits. The audience is the global Rails community. (The runtime report is the exception: it speaks the user's language.)
- **Naming:** "Nuke on Rails" in prose; `nuke-on-rails` in code/paths; never `nukeonrails`.

## Code of conduct

Be kind. Disagree on technical merit. Rails has been a community since 2004 — let's keep this feeling like that.
