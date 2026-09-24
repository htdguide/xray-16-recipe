# src/xrGame/stalker_kill_wounded_planner.h

> Declares the sub-planner that executes a downed enemy.

**Needs** — [`stalker_kill_wounded_planner.cpp`](stalker_kill_wounded_planner.cpp.md) · [`action_planner_action_script.h`](action_planner_action_script.h.md) · [`ai/stalker/ai_stalker.h`](ai/stalker/ai_stalker.h.md)
**Used by** — [`stalker_combat_planner.cpp`](stalker_combat_planner.cpp.md) · [`stalker_kill_wounded_planner.cpp`](stalker_kill_wounded_planner.cpp.md)
**Tier floor** — T2: a five-operator planner that is itself an operator.

## Purpose

Declares the surface implemented in
[`stalker_kill_wounded_planner.cpp`](stalker_kill_wounded_planner.cpp.md). It is the only
sub-planner that overrides every lifecycle hook, because it has to tell the combat planner
above it that an execution is in progress.

## Exported units

- `setup(creature, parent_property_storage)` — rebuild the tables.
- `initialize()` — branch entry: reset progress and raise the flag the parent reads.
- `execute()` / `update()` — pure delegation.
- `finalize()` — branch exit: lower that flag and restore the creature's alertness.
- `add_evaluators()` / `add_actions()`.
