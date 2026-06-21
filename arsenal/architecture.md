# Weapon: Architectural Boundaries

The structural rules that keep layers from bleeding into each other — dependency *direction*, not dependency *count*. rubycritic measures the complexity inside a file; it never tells you a model reached up into a controller, or that two namespaces quietly became one unit. This weapon reads the **visible edges** between layers — constant references, inheritance, `include`/`extend`, method sends and definitions — and flags the ones that point the wrong way.

Distilled from the techniques of the **archspec** gem. The gem enforces an architecture the team *declared* in a DSL; this weapon *infers* the violations by reading source, so it works on a repo with no declared spec. The one archspec rule that does **not** translate is "this directory must stay empty" (`must_be_empty`) — banning a whole category of object depends on a convention the team declared, which a zero-config audit can't know. Everything below is detectable without a spec.

Apply it to the `app/` layer folders (`models`, `controllers`, `services`, `jobs`, `lib/`) and the hotspot files. This overlaps `arsenal/code-quality.md` on "logic in the wrong layer" — that file owns *altitude and simplification*; this one owns *direction and cycles*.

## Dependency direction (a layer pointing the wrong way)

The web layer may depend on the domain; the domain must not depend on the web. A file under `app/models`, `app/services`, `app/jobs`, or `lib/` that references `params`, `session`, `request`, `cookies`, `flash`, `render`, `redirect_to`, a route helper (`*_path`/`*_url`), a `*Controller`, or `ActionController`/`ActionView` has the dependency arrow upside down — the domain now can't be tested or reused without the web stack.

```ruby
# Problem — a model reaches up into the web layer: it reads params and touches flash
class Order < ApplicationRecord
  def process!(params)
    self.status = params[:status]                 # a controller concern leaking into the model
    ApplicationController.helpers.flash[:notice] = "done"
  end
end

# Fix — keep the model ignorant of the web; the controller passes plain data in
class Order < ApplicationRecord
  def process!(status:)
    update!(status: status)
  end
end
```

## Dependency cycles (two layers that became one)

When namespace A references B and B references A, the two are a single unit in practice: you can't change, test, or extract one without the other. Look for mutual constant references between two namespaces, modules, or concerns.

```ruby
# Problem — Billing and Catalog reference each other: two namespaces fused into one
module Billing
  def self.price_for(sku) = Catalog::Product.find(sku).price   # Billing -> Catalog
end
module Catalog
  class Product < ApplicationRecord
    def invoice = Billing::Invoice.for(self)                    # Catalog -> Billing
  end
end

# Fix — make the dependency point one way: extract the shared concept, or have Catalog emit a
# plain value that Billing consumes, and never call back into Billing.
```

## One-shot object ceremony

A class that exists only to be instantiated and immediately invoked — `Foo.new(x).call` with the instance never used again — is a function wearing a class costume: extra ceremony, a throwaway object per call, and a constructor that's really just an argument list. (Opinionated — flag it as a smell, not a defect; pairs with `arsenal/code-quality.md`.)

```ruby
# Problem — built only to be called once; the instance has no other purpose
total = PriceCalculator.new(cart).call

# Fix — a module function (no instance to manage), or a plain method on an object that has state
total = PriceCalculator.call(cart)

module PriceCalculator
  module_function
  def call(cart) = cart.items.sum(&:price)
end
```

## Inconsistent component interface (protocol drift)

A folder of service/query/command objects where some entry points are `call`, some `perform`, some `run` has no contract — every caller has to guess the verb, and nothing can iterate over them generically. Pick one entry-point name per component and hold it.

```ruby
# Problem — a folder of service objects with no shared entry point
CreateUser.new(attrs).call
ChargeCard.new(order).perform     # different verb
SendInvoice.new(order).run        # different again

# Fix — one entry-point name across the component
CreateUser.call(attrs)
ChargeCard.call(order)
SendInvoice.call(order)
```

## Zeitwerk name/path drift

Rails autoloads by convention: `app/services/billing/charge.rb` must define `Billing::Charge`. When the file path and the constant disagree, autoloading raises in some load orders and works in others — a heisenbug — and every reader is misled about where a constant lives. Compute the expected constant from the path and confirm the file defines it.

```ruby
# Problem — path and constant disagree → NameError / LoadError under some load orders
# app/services/billing/charge.rb
class ChargeService    # Zeitwerk expects Billing::Charge here
end

# Fix — name the constant to match the path (or move the file to match the constant)
# app/services/billing/charge.rb
module Billing
  class Charge
  end
end
```

## Severity and remedies

These are **maintainability-tier** findings (quality, OWASP A06 territory), reported alongside `arsenal/code-quality.md` — above cosmetic nits, below any security finding. Within the tier, rank by how much a violation *freezes the codebase*:

- **Dependency cycles and upside-down direction** are the high-value findings: they're why a "small" refactor turns into a week. State the two layers and the edge that shouldn't exist.
- **Zeitwerk drift** can be a real bug (autoload failure), not just style — rank it higher when you can show a load-order it breaks.
- **One-shot ceremony and protocol drift** are softer style findings — group them, don't lead with them.

Remedies: invert the dependency (the domain emits data, the web consumes it); extract the shared concept to break a cycle; move web concerns back to the controller and pass plain data into the domain; rename the file or constant to match the Zeitwerk path; pick one entry-point name per component. And the named tool: when the team wants these boundaries *enforced going forward*, the **archspec** gem codifies exactly these rules (`can_use`/`cannot_use`, `no_cycles!`, `verify_zeitwerk_names!`) and fails the build on the next violation — the structural lock, the same way strong_migrations locks migration safety.
