# Weapon: Code Quality (Thermo-Nuclear)

The quality engine of Nuke on Rails — absorbed from the Thermo-Nuclear Code Quality Review and translated to the Rails idiom. Apply this weapon to the hotspot files selected by the churn × complexity quadrant, never to the whole codebase uniformly.

Above all, be **ambitious** about code structure. Do not merely identify local cleanup opportunities. Actively search for "code judo" moves: restructurings that preserve behavior while making the implementation dramatically simpler, smaller, more direct, and more elegant.

## Core Prompt

Start from this baseline:

> Perform a deep code quality audit of the hotspot files.
> Rethink how each hotspot could be structured to meaningfully improve code quality without impacting behavior.
> Work to improve abstractions, modularity, reduce spaghetti code, improve succinctness and legibility.
> Be ambitious: if there is a clear path to improving the implementation that involves restructuring part of the codebase, propose it.
> Be extremely thorough and rigorous. Measure twice, cut once.

## Non-Negotiable Standards

0. **Be ambitious about structural simplification.**
   - Do not stop at "this could be a bit cleaner."
   - Look for opportunities to reframe the design so that whole branches, helpers, modes, conditionals, or layers disappear entirely.
   - Prefer the solution that makes the code feel inevitable in hindsight.
   - Assume there is often a "code judo" move available: a re-organization that uses the existing architecture more effectively and makes the code dramatically simpler.
   - If you see a path to delete complexity rather than rearrange it, push hard for that path.

1. **Treat any file over 1,000 lines as a strong code-quality smell by default.**
   - Prefer extracting POROs, service objects, query objects, or focused modules over letting a file sprawl.
   - Only waive this if there is a compelling structural reason and the file is still clearly organized.

2. **Do not tolerate spaghetti branching.**
   - Be highly suspicious of ad-hoc conditionals, scattered special cases, and one-off branches inserted into unrelated flows.
   - "Weird if statements in random places" are a design problem, not a stylistic nit.
   - Prefer pushing the logic into a dedicated abstraction: a policy object, a state machine, a form object, a separate module.

3. **Bias toward cleaning the design, not just accepting working code.**
   - If behavior can stay the same while the structure becomes meaningfully cleaner, push for the cleaner version.
   - Do not rubber-stamp "it works" implementations that leave the codebase messier.
   - Strongly prefer simplifications that remove moving pieces altogether over refactors that merely spread the same complexity around.

4. **Prefer direct, boring, maintainable code over hacky or magical code.**
   - Treat brittle, ad-hoc, or "magic" behavior (heavy metaprogramming, `method_missing`, dynamic sends without need) as a code-quality problem.
   - Flag thin abstractions, identity wrappers, or pass-through helpers that add indirection without buying clarity.

5. **Push hard on boundary cleanliness.**
   - Question hash-shaped data passed across layers when an explicit object (PORO, value object, form object) would make the contract clear.
   - Be suspicious of long `&.` chains and nil-tolerant code that papers over an unclear invariant — ask whether the boundary should guarantee presence instead.
   - Prefer explicit models or shared contracts over loosely-shaped ad-hoc hashes.

6. **Keep logic in the canonical Rails layer and reuse existing helpers.**
   - Business logic belongs in models or dedicated objects — not in controllers, views, helpers, or serializers.
   - Call out feature logic leaking into shared paths, and bespoke one-offs where the codebase already has a canonical utility.
   - Push code toward the layer that already owns the concept instead of normalizing architectural drift.

7. **Treat unnecessary sequential orchestration and non-atomic updates as design smells.**
   - Related writes that can leave state half-applied belong in a transaction.
   - Slow work in the request cycle that belongs in a background job is a structural finding, not a nit.
   - Do not over-index on micro-optimizations, but flag avoidable orchestration complexity that makes the implementation brittle.

8. **Require test coverage for business rules, authorization, and sensitive data.**
   - Every business rule must have a test covering its critical cases.
   - Permissions and authorization checks must be tested — untested authorization is a security finding, not just a quality finding (escalate it to the security side of the report).
   - Sensitive or critical data handling must have explicit coverage.

## Rails Translation

The classic smells of this weapon, in their Rails form:

- **Fat models**: god objects (usually `User`) accumulating every concern in the app. The fix is extraction into POROs, service objects, or domain modules — not a `concerns/` folder that hides the same mess.
- **Rug concerns**: `app/models/concerns/` used as a rug to sweep code under. A concern that is only included in one model is a file split pretending to be an abstraction.
- **Service-layer ceremony**: the over-extraction smell — a `*Service` whose `call` just wraps one model method, adding indirection with zero capability — and its opposite, the god service (`OrderService`/`*Manager`/`*Handler` accumulating a dozen unrelated verbs: a fat model that escaped to `app/services/`). Logic *earns* a service only when it orchestrates 3+ models in one transaction, calls an external API, has multiple outcomes a caller must branch on, or has no model home — not by default.

  ```ruby
  # Problem — a service that wraps a single model method: a class, a file, and zero capability
  class DeletePostService
    def initialize(post) = @post = post
    def call = @post.destroy!
  end
  DeletePostService.new(post).call

  # Fix — it's a model method
  post.destroy!
  ```
- **Callback-driven business logic**: chains of `before_save`/`after_commit` that implement workflows invisibly. Side effects belong in an explicit object the caller invokes, not in hooks that fire by surprise.

  ```ruby
  # Problem — a workflow hidden in a callback: creating a User silently emails and charges
  class User < ApplicationRecord
    after_create :send_welcome_email, :charge_signup_fee   # fires on every create — tests, imports, admin
  end

  # Fix — make the side effects an explicit step the caller invokes
  class RegisterUser
    def call(attrs) = User.create!(attrs).tap { |user| Onboarding.new(user).run }
  end
  ```
- **Controller bloat**: business decisions, queries, and orchestration living in actions. Controllers should authenticate, authorize, delegate, and respond.
- **Logic in views and helpers**: conditionals and data shaping in ERB or in grab-bag helper modules.
- **N+1 and query leakage**: queries scattered through views and serializers; missing `includes`; `.all` loaded into memory to filter in Ruby. Two variants worth naming: a **`.count` inside a loop** (`post.comments.count` per row → a `SELECT COUNT(*)` each; the fix is `counter_cache` + `.size`), and a **raw query inside a loop** (`User.find_by(...)` per element, no association — the kind association-aware detection misses).

  ```ruby
  # Problem — N+1: one query for the posts, then one more per post for its author (1 + N queries)
  @posts = Post.all
  # in the view: <% @posts.each do |post| %> <%= post.author.name %> <% end %>

  # Fix — eager-load the association: two queries total, no matter how many posts
  @posts = Post.includes(:author)
  ```

  A static read finds the *shape* (a loop over a collection that touches an association with no eager-load), not the query *count* — report it as "covers, doesn't guarantee". The durable remedy is detection in CI: **Bullet** (dev + test, `raise` on N+1) plus **Prosopite** for the raw-query-in-a-loop case Bullet can't see.
- **Scope/SQL drift**: raw SQL strings duplicating what scopes already express, or scopes so complex they hide an unindexed query.
- **`default_scope`**: it silently rewrites every query on the model and breaks expectations the moment someone needs `unscoped`. Treat any non-trivial `default_scope` as a finding.
- **Swallowed exceptions**: `rescue Exception`, bare `rescue` returning nil, or rescue blocks that log nothing — failure hidden to "make it work" is debt with interest, and a hallmark of unreviewed AI-generated code.

  ```ruby
  # Problem — failure hidden to "make it work": the charge can fail and the order still ships
  def checkout
    Payment.charge!(order)
  rescue => e   # swallowed: nothing logged, nothing re-raised
    nil
  end

  # Fix — handle the specific error, record it, and let the rest fail loudly
  def checkout
    Payment.charge!(order)
  rescue Payment::Declined => e
    Rails.logger.warn("charge declined: #{e.message}")
    raise
  end
  ```
- **Request state in class/global variables (thread-unsafe)**: Rails serves requests on multiple threads (Puma) and runs jobs threaded, so a class variable (`@@current_user`), a class-level `attr_accessor`, a mutated constant, or memoization on the class that holds request-scoped data is shared across threads — one request's data leaks into another's (a cross-user leak, like the cache and broadcast leaks in `arsenal/authorization.md`) or races. Keep per-request state on `ActiveSupport::CurrentAttributes` or controller instance variables, never on the class.

  ```ruby
  class Tenant
    @@current = nil                # Problem — shared across threads; one request's tenant leaks to another
    def self.current=(t) = @@current = t
  end
  class Current < ActiveSupport::CurrentAttributes  # Fix — per-request, thread-isolated, reset each request
    attribute :tenant
  end
  ```

## Primary Review Questions

For every hotspot file, ask:

- Is there a "code judo" move that would make this dramatically simpler?
- Can this design be reframed so fewer concepts, branches, or helper layers are needed?
- Did a previously cohesive module become coupled, stateful, or hard to scan?
- Is this logic living in the right file and layer?
- Are there repeated conditionals that signal a missing model or missing object?
- Is the implementation direct and legible, or does it rely on special cases and incidental control flow?
- Is each abstraction actually earning its keep, or is it just a wrapper?
- Does data cross boundaries as loose hashes where an explicit object would make the invariant clear?
- Is orchestration more sequential or less atomic than it needs to be?
- Do the business rules in this file have tests covering their critical cases? Are authorization checks tested?

## What to Flag Aggressively

- A complicated implementation where a cleaner reframing could delete whole categories of complexity.
- Refactors that move code around but fail to reduce the number of concepts a reader must hold in their head.
- Files past 1,000 lines, especially when cohesive pieces could be split out.
- One-off booleans, nullable modes, or flags that complicate existing control flow.
- Feature-specific logic leaking into general-purpose modules.
- Magic metaprogramming that hides simple structure.
- Thin wrappers or identity abstractions that add indirection without simplifying anything.
- Copy-pasted logic instead of extracted helpers.
- Edge-case handling implemented in the middle of an already busy method.
- "Temporary" branching that has clearly become permanent debt.
- Bespoke helpers where the codebase already has a canonical utility for the job.
- Callback chains implementing business workflows.
- Sequential flow where obviously independent work could be simpler and clearer run in parallel (or in a background job).
- Partial-update logic that leaves state less atomic than necessary.
- Business rules, permissions, or sensitive-data handling without test coverage.

## Preferred Remedies

- Delete a whole layer of indirection rather than polishing it.
- Reframe the state model so conditionals disappear instead of getting centralized.
- Change the ownership boundary so the feature becomes a natural extension of an existing abstraction.
- Turn special-case logic into a simpler default flow with fewer exceptions.
- Extract a PORO, service object, query object, form object, or pure function.
- Split a large file into smaller focused modules — by domain concept, not by line count.
- Replace condition chains with an explicit model or dispatcher.
- Separate orchestration from business logic; wrap related writes in a transaction.
- Move callback side effects into an explicit object the caller invokes.
- Collapse duplicate branches into a single clearer flow.
- Reuse the existing canonical helper instead of introducing a near-duplicate.
- Restructure related updates into a more atomic flow when partial state would be harder to reason about.
- Add tests before touching business rules that have no coverage.

Do not be satisfied with "maybe rename this" feedback when the real issue is structural.
Do not be satisfied with a merely cleaner version of the same messy idea if there is a plausible path to a much simpler idea.

## Severity Bar

Treat these as presumptive top-severity quality findings unless the code clearly justifies them:

- A lot of incidental complexity preserved where there is a plausible code-judo move that would delete it.
- A file well past 1,000 lines whose cohesive pieces could clearly be split out.
- Ad-hoc branching that makes a core flow tangled, or feature checks scattered across shared code.
- An unnecessary abstraction, wrapper, or hash-shaped contract that makes the design more indirect than the direct flow.
- A duplicated helper, or logic living in the wrong layer when there is a clear canonical home.
- Business rules or authorization without test coverage (authorization gaps also go to the security side).

Do not approve a hotspot as "fine" merely because behavior seems correct. The bar is: no clear structural problem, no obvious missed opportunity for a dramatic simplification, no unjustified file-size explosion, no spaghetti branching, no magic that obscures the design, and critical rules under test.

## Tone and Output

Be direct, serious, and demanding about quality. Do not be rude, but do not soften major maintainability issues into mild suggestions. If the code is messy, say so clearly; if there is a dramatic simplification available, say that clearly too.

Phrase findings the way a principal engineer would in review:

- `this file is past 1k lines and mixes three domains. decompose it first — here's the seam.`
- `this is another special-case branch in an already busy flow. move it behind its own abstraction.`
- `this works, but it makes the surrounding code more spaghetti. keep the behavior, restructure the implementation.`
- `this looks like feature logic leaking into a shared path. isolate it.`
- `this abstraction isn't earning its keep. keep the direct flow.`
- `this callback chain implements a workflow invisibly. make it an explicit object the caller invokes.`
- `this is a bespoke helper for something the codebase already has. reuse the canonical one.`
- `there's a code-judo move here: reframe the model and these branches disappear.`
- `this refactor moves complexity around but doesn't delete it. the model itself can be simpler.`
- `this business rule has no test covering its critical cases.`

Prioritize findings in this order:

1. Structural code-quality problems in high-churn files
2. Missed opportunities for dramatic simplification / code-judo restructuring
3. Spaghetti / branching complexity
4. Boundary and abstraction problems that make the code harder to reason about
5. File-size and decomposition concerns
6. Missing test coverage on business rules and authorization (escalate authorization gaps to the security side)
7. Legibility and maintainability concerns

Do not flood the report with low-value nits if there are larger structural issues. Prefer a small number of high-conviction findings over a long list of cosmetic notes. Every finding feeds the unified impact-ranked report — give it a severity, a location, and a concrete first step.
