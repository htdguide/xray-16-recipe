# src/xrAICore/Components/script_world_state_script.cpp

> Publishes the planner's world state to the script layer under the name `world_state`, so that shipped scripts can build the goals the planner is asked to satisfy.

**Needs** — [`condition_state.h`](condition_state.h.md) · [`../Navigation/graph_engine_space.h`](../Navigation/graph_engine_space.h.md) · [Seam: Script binding layer](../../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: a declarative registration of an existing type into the script virtual machine.

## Purpose

The goal a creature is planning toward is authored in script, which means the partial world
state must be constructible and editable from script. This file is that boundary, and its
naming is frozen by the shipped scripts.

## `world_state` (script class)

**Contract** — registers the planner's state type with:

- a default constructor, producing the empty state — which means "no requirements"
- a copy constructor
- `add_property(property)` — insert a property/value pair, preserving the ordering invariant;
  the script-visible name differs from the native one, which says *condition*
- `remove_property(property_id)` — drop the pair for a property
- `clear()` — back to the empty state
- `includes(other)` — does this state satisfy every requirement in `other`
- `property(property_id)` — look one pair up
- ordering and equality, matching the native ones

**Notes** — the script surface exposes the mutating half of the state and the two query
routines, and deliberately not the hash or the difference operator; those are internal to
planning.

Two native traps are reachable from script through this surface and a rebuild should close
both rather than reproduce them. Inserting a property the state already has is a hard failure
rather than a replace, so a script that re-asserts a goal property crashes the engine rather
than being ignored. And the lookup returns the next property at or after the requested one
when the requested one is absent, so a script reading a property it did not set gets a
different property's value — see
[`condition_state_inline.h`](condition_state_inline.h.md).
