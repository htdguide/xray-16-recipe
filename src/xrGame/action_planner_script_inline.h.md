# src/xrGame/action_planner_script_inline.h

> Binds a script-facing brain to a concrete creature by deriving the script facade from it.

**Needs** — [`action_planner_script.h`](action_planner_script.h.md) · [`action_planner.h`](action_planner.h.md)
**Used by** — [`action_planner_script.h`](action_planner_script.h.md)
**Tier floor** — T3: two assignments

## Purpose

The simplest of the three bridges, and the only one that goes in one direction only: a
planner is always set up by engine code that already holds the concrete creature, never by
the script layer handing it a facade. So there is no recovery step and no downcast — only
the derivation.

## State

Adds the concrete creature alongside the base planner's facade binding.

Invariant: both name the same entity, or both are absent; the facade is never set from
anywhere but here.

## `setup`

**Contract** — requires a concrete creature, sets the base planner up against that
creature's script facade, and records the creature. Everything the base's setup does —
clearing the world-state storage, resetting the running action — happens as usual.

```text
FUNCTION setup(creature)
  REQUIRE creature present
  base.setup(creature.script_facade)
  concrete = creature
```

## `object`

**Contract** — the concrete creature. Unlike its two sibling bridges this accessor does
*not* check the binding, so reading it before setup yields an absent reference. A rebuild
should check it; the asymmetry is an oversight.
