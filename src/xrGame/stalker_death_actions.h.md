# src/xrGame/stalker_death_actions.h

> Declares the action that runs while a stalker is dying.

**Needs** — [`stalker_death_actions.cpp`](stalker_death_actions.cpp.md) · [`stalker_base_action.h`](stalker_base_action.h.md)
**Used by** — [`stalker_death_actions.cpp`](stalker_death_actions.cpp.md) · [`stalker_death_planner.cpp`](stalker_death_planner.cpp.md)
**Tier floor** — T2: one action object per creature.

## Purpose

Declares the surface implemented in
[`stalker_death_actions.cpp`](stalker_death_actions.cpp.md). One class, with one private
predicate deciding whether the corpse should fire off the rest of its magazine.

## Exported units

- `Dead` — entry and per-cycle. There is no exit hook: the action ends only when the
  creature does.
