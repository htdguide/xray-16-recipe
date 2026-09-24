# src/xrGame/stalker_danger_by_sound_planner.h

> Declares the sub-planner for a sound-only threat.

**Needs** — [`stalker_danger_by_sound_planner.cpp`](stalker_danger_by_sound_planner.cpp.md) · [`action_planner_action_script.h`](action_planner_action_script.h.md) · [`ai/stalker/ai_stalker.h`](ai/stalker/ai_stalker.h.md)
**Used by** — [`stalker_danger_by_sound_planner.cpp`](stalker_danger_by_sound_planner.cpp.md) · [`stalker_danger_planner.cpp`](stalker_danger_planner.cpp.md)
**Tier floor** — T2: a one-operator planner that is itself an operator.

## Purpose

Declares the surface implemented in
[`stalker_danger_by_sound_planner.cpp`](stalker_danger_by_sound_planner.cpp.md), which
documents why this branch is unreachable.

## Exported units

- `setup(creature, parent_property_storage)` — rebuild the tables.
- `initialize()` / `update()` / `finalize()` — pure delegation.
- `add_evaluators()` / `add_actions()`.
