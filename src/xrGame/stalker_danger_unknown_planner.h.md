# src/xrGame/stalker_danger_unknown_planner.h

> Declares the sub-planner for a threat with no known direction.

**Needs** — [`stalker_danger_unknown_planner.cpp`](stalker_danger_unknown_planner.cpp.md) · [`action_planner_action_script.h`](action_planner_action_script.h.md) · [`ai/stalker/ai_stalker.h`](ai/stalker/ai_stalker.h.md)
**Used by** — [`stalker_danger_planner.cpp`](stalker_danger_planner.cpp.md) · [`stalker_danger_unknown_planner.cpp`](stalker_danger_unknown_planner.cpp.md)
**Tier floor** — T2: a three-operator planner that is itself an operator.

## Purpose

Declares the surface implemented in
[`stalker_danger_unknown_planner.cpp`](stalker_danger_unknown_planner.cpp.md).

## Exported units

- `setup(creature, parent_property_storage)` — rebuild the tables.
- `initialize()` — branch entry: release the cover claim and reset progress.
- `update()` / `finalize()` — pure delegation; present only because the base declares them.
- `add_evaluators()` / `add_actions()`.
