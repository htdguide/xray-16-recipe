# src/xrGame/stalker_death_planner.h

> Declares the death branch of a stalker's brain.

**Needs** — [`stalker_death_planner.cpp`](stalker_death_planner.cpp.md) · [`action_planner_action_script.h`](action_planner_action_script.h.md) · [`ai/stalker/ai_stalker.h`](ai/stalker/ai_stalker.h.md)
**Used by** — [`stalker_death_planner.cpp`](stalker_death_planner.cpp.md) · [`stalker_planner.cpp`](stalker_planner.cpp.md)
**Tier floor** — T2: a two-operator planner that is itself an operator.

## Purpose

Declares the surface implemented in
[`stalker_death_planner.cpp`](stalker_death_planner.cpp.md).

## Exported units

- `setup(creature, parent_property_storage)` — rebuild the tables.
- `add_evaluators()` / `add_actions()`.

It overrides neither `update` nor `initialize`: a dead creature's branch has nothing to
publish upward and nothing to reset.
