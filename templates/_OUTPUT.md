# Report output — house structure

The canonical shape of the Step 5 report. **Pairs with `SKILL.md` Step 5**, which governs *what goes in* and *how it's ranked*; this file governs *how it looks*.

The report is rendered to a **TERMINAL**, so it uses **no markdown** — no `#` headings, `**bold**`, `| tables |`, or `>` quotes; they print as literal junk. Hierarchy comes from divider rules, `▸`-anchored section titles, the hollow `▹` on finding fields, tree branches (`├─ └─`), and the severity emoji.

Write the whole report in the **user's language**. The English below is the format demo only — the labels (`Problem` / `Solution` / `Files` / `Technical details`, `Scan`, `TL;DR`…) get translated too.

Layout rules:
- Everything flush **LEFT** — never indent inward (the reader copies straight out of the terminal).
- Prose runs **continuous** — no manual line breaks mid-paragraph; let the terminal wrap.
- Lists use **tree branches** (`├─` for items, `└─` to close), flush left.
- **Dividers:** the heavy `════` frames the report and closes the banner; a light `────` separates the scoreboard, **each finding**, and each closing block. Every block gets air.
- **Spacing after a `▸` title:** a title whose body is a **tree** hugs it (the branches connect upward — Stack, Scan, Fix next). The **inline scoreboard** and the **numbered fix list** get a blank line after the title so they breathe. Bare prose blocks (TL;DR, the structural win, notes) put their text on the very next line.
- **`▸` prefixes a section title** (Stack, Scan, scoreboard, the fix lists, the discarded line). Text-only triangle U+25B8 — never the emoji `▶`. Bare prose blocks read without it.
- **Inside a finding, the field labels use the hollow `▹`** (U+25B9, text-only): `▹ Problem:`, `▹ Solution:`, `▹ Files:`, and the optional `▹ Technical details:`. It rhymes with `▸` but is hollow — section above, field below. `Problem`/`Solution`/`Technical details` put their prose on the line below; `Files` hugs its tree.
- **The title is just `<emoji> <LEVEL> — <plain headline>` — nothing else.** No finding number, no effort estimate, no metrics; the headline is plain language. Cross-references name the finding by subject ("the Strava endpoint"), not a number.
- **Plain language first.** `Problem` and `Solution` speak to a human — a non-expert stakeholder and an exhausted senior both understand them in one pass. Method names, line refs, exploit steps, and any code metrics (churn × complexity, grade, smell counts) go in `Technical details`, which is optional and comes last.
- **The scoreboard opens the body**, right after the banner, as one inline line — count before the label (`🔴 1 Critical  🟠 4 High …`). Dependency-risk totals and any end-of-life flag stay in the banner (Stack/Scan); never repeat them here.
- **The "ruled out" line is ONE compact line near the end** — not a verbose section. It exists so the reader knows the scanner noise was checked, not missed; keep it to a sentence.
- **The structural win is ONE tight sentence** — the single change that removes the most risk at once. Never a paragraph.
- **No effort or time estimates.** How long a fix takes depends on the team and their tooling — an AI can collapse a "multi-day" refactor into minutes — so the report never guesses; the fix list conveys *scope* in plain words.
- **Emoji only where it carries signal:** the 🔴🟠🟡🟢 severity marker on each finding title and in the scoreboard. Nowhere else (the ☢️ in the title and the ⚠️ on the confidentiality line are the two fixed exceptions).

The shape (this block is the demo — render it in the user's language):

```text
════════════════════════════════════════════════════════════════════════════

☢️  NUKE ON RAILS — <codename>

▸ Stack
├─ <Rails x.y.z — flag EOL / no security support if applicable>
└─ <Ruby x.y.z — note substitution if engines ran under another Ruby>

▸ Scan
├─ <N Brakeman alerts>
├─ <N advisories / M gems (note where they concentrate)>
└─ <rubycritic score /N modules — churn-quadrant reliability if relevant>

TL;DR
<2–3 continuous sentences. Dry, deadpan wit in the framing; findings stay sober.
The joke is written natively in the user's language — never a translated one.>

⚠️  Confidential report (do not publish)

════════════════════════════════════════════════════════════════════════════

▸ SCOREBOARD

🔴 <n> Critical   🟠 <n> High   🟡 <n> Medium   🟢 <n> Low

────────────────────────────────────────────────────────────────────────────

<emoji>  <LEVEL> — <headline: what & where, in plain language>

▹ Problem:

<plain, human language — what's broken and why it matters, in everyday terms. A non-expert
and a tired senior both get it in one read. No method names, line refs or jargon here.>

▹ Solution:

<plain, human language — what to do, conceptually.>

▹ Files:
├─ <file:line>
└─ <file:line>

▹ Technical details:   (optional — only when there's a code path, exploit, or metric worth showing)

<the precise specifics: method names, line refs, the reachable path, the exploit steps;
for a quality hotspot, the churn × complexity, grade and smell counts.>

────────────────────────────────────────────────────────────────────────────

<next finding, same shape — each separated from the next by this light rule.
Order by impact: confirmed exploitable security → reachable CVEs → high-churn ×
complexity hotspots → theoretical security → remaining quality.>

────────────────────────────────────────────────────────────────────────────

▸ FIX NOW (highest leverage first)

1. <subject> — <the move, one tight line>
2. <subject> — <…>

▸ FIX NEXT

1. <subject> — <the move, one tight line>
2. <subject> — <…>

────────────────────────────────────────────────────────────────────────────

BIGGEST STRUCTURAL MULTIPLIER
<one tight sentence: the single structural change that removes the most risk at once.>

────────────────────────────────────────────────────────────────────────────

▸ RULED OUT

<one compact line: the scanners flagged these, you checked, they're fine — name them with a
one-word reason each (false positive / not reachable / dev-only / wrong target).>

────────────────────────────────────────────────────────────────────────────

NOTES
<operational honesty: Ruby/engine substitutions, no artifacts left in the repo,
"the arsenal covers these areas, it doesn't guarantee them". Omit if nothing applies.>

════════════════════════════════════════════════════════════════════════════
```

<!--
  Checklist before this template ships / changes (delete this comment in usage):
  - Terminal-safe: no markdown. Plaintext + box-drawing only. Flush left, continuous prose.
  - Heavy ════ frames the report + closes the banner; light ──── between scoreboard,
    each finding, and each closing block.
  - Tree-bodied ▸ titles hug; inline scoreboard and numbered fix lists get a blank line.
  - Scoreboard opens the body as ONE inline line (count before label). Dep-risk/EOL only in banner.
  - Title is just emoji + LEVEL + plain headline — no number, no effort, no metrics.
  - ▸ (U+25B8, filled) on section titles; ▹ (U+25B9, hollow) on finding fields. Neither is ▶.
  - Problem/Solution in plain human language; jargon, exploit path and metrics in Technical details.
  - "Ruled out" is ONE compact line near the end; structural win is ONE sentence.
  - No time/effort estimates; fix lists are numbered, parallel, by subject.
  - Emoji only on severity (finding title + scoreboard); ☢️ title and ⚠️ line aside.
  - Rendered in the user's language at runtime; stays in sync with SKILL.md Step 5.
-->
