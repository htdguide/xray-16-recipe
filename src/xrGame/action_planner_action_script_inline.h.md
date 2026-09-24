# src/xrGame/action_planner_action_script_inline.h

> Recovers the concrete creature from the script-visible facade when a composite engine action is installed into a script-facing planner.

**Needs** — [`action_planner_action_script.h`](action_planner_action_script.h.md) · [`script_game_object.h`](script_game_object.h.md)
**Used by** — [`action_planner_action_script.h`](action_planner_action_script.h.md)
**Tier floor** — T2: a downcast per installation

## Purpose

Identical in intent to [`action_script_base_inline.h`](action_script_base_inline.h.md),
applied to the composite action: the facade and the concrete object are always derived
from one another so they cannot disagree, in both directions.

## State

Adds the concrete creature alongside the base's facade binding.

Invariant: both name the same entity, or the type refuses to be set up.

## Construction

**Contract** — takes the concrete creature, optionally with an initial precondition and
effect set, and a diagnostic name; passes the derived facade up and records the creature.

## `setup`

**Contract** — the planner calls the facade-taking overload. It runs the composite's own
setup (which binds both halves and aims the inner plan at this action's effects), recovers
the concrete creature from the facade, requires the recovery to have succeeded, and calls
the object-taking overload that subclasses override.

**Invariants** — unlike the leaf-action bridge, the narrowing here is *checked*: a facade
whose underlying object is not of the expected type fails at installation rather than
leaving an absent binding for a later assertion to catch. A rebuild should apply this
stricter behaviour to both bridges.

```text
FUNCTION setup(facade, storage)
  REQUIRE facade present
  composite.setup(facade, storage)
  concrete = facade.underlying_object AS concrete type
  REQUIRE concrete present
  setup(concrete, storage)              # subclass hook

FUNCTION setup(object, storage)         # default: assert both bindings, change nothing
  REQUIRE object present AND concrete present
```

**Notes** — the default object-taking overload deliberately does *not* re-assign the
concrete binding; the facade-taking one already did. Its body is two assertions, which
makes it a contract check rather than an operation.

## `object`

**Contract** — the concrete creature; requires it to be bound.
