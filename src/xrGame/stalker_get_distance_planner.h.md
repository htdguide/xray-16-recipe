# src/xrGame/stalker_get_distance_planner.h

> Declares the sub-planner for closing on an enemy that is out of effective range.

**Needs** — [`stalker_get_distance_planner.cpp`](stalker_get_distance_planner.cpp.md) · [`action_planner_action_script.h`](action_planner_action_script.h.md) · [`ai/stalker/ai_stalker.h`](ai/stalker/ai_stalker.h.md)
**Used by** — [`stalker_combat_planner.cpp`](stalker_combat_planner.cpp.md) · [`stalker_get_distance_planner.cpp`](stalker_get_distance_planner.cpp.md)
**Tier floor** — T2: a two-operator planner that is itself an operator.

## Purpose

Declares the surface implemented in
[`stalker_get_distance_planner.cpp`](stalker_get_distance_planner.cpp.md).

## Exported units

- `setup(creature, parent_property_storage)` — rebuild the tables and drop the cached cover.
- `add_evaluators()` / `add_actions()`.

It overrides no per-cycle hook: the loop between its two actions is driven entirely by
their preconditions.
