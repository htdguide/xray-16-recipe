# src/xrGame/stalker_danger_grenade_planner.h

> Declares the sub-planner for a live grenade.

**Needs** — [`stalker_danger_grenade_planner.cpp`](stalker_danger_grenade_planner.cpp.md) · [`action_planner_action_script.h`](action_planner_action_script.h.md) · [`ai/stalker/ai_stalker.h`](ai/stalker/ai_stalker.h.md)
**Used by** — [`stalker_danger_grenade_planner.cpp`](stalker_danger_grenade_planner.cpp.md) · [`stalker_danger_planner.cpp`](stalker_danger_planner.cpp.md)
**Tier floor** — T2: a five-operator planner that is itself an operator.

## Purpose

Declares the surface implemented in
[`stalker_danger_grenade_planner.cpp`](stalker_danger_grenade_planner.cpp.md).

## Exported units

- `setup(creature, parent_property_storage)` — rebuild the tables.
- `initialize()` — branch entry: reset the two progress propositions.
- `update()` / `finalize()` — pure delegation.
- `add_evaluators()` / `add_actions()`.
