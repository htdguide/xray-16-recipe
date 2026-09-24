# src/xrGame/stalker_low_cover_planner.h

> Declares the sub-planner for fighting from cover that only protects a crouched creature.

**Needs** — [`stalker_low_cover_planner.cpp`](stalker_low_cover_planner.cpp.md) · [`action_planner_action_script.h`](action_planner_action_script.h.md) · [`ai/stalker/ai_stalker.h`](ai/stalker/ai_stalker.h.md)
**Used by** — [`stalker_combat_planner.cpp`](stalker_combat_planner.cpp.md) · [`stalker_low_cover_planner.cpp`](stalker_low_cover_planner.cpp.md)
**Tier floor** — T2: a three-operator planner that is itself an operator.

## Purpose

Declares the surface implemented in
[`stalker_low_cover_planner.cpp`](stalker_low_cover_planner.cpp.md).

## Exported units

- `setup(creature, parent_property_storage)` — rebuild the tables.
- `initialize()` — branch entry: pin the creature in place.
- `update()` — refresh the cover hint, then run a cycle.
- `execute()` / `finalize()` — pure delegation.
- `add_evaluators()` / `add_actions()`.
