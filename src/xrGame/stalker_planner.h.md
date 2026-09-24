# src/xrGame/stalker_planner.h

> Declares the root planner of a stalker's brain.

**Needs** — [`stalker_planner.cpp`](stalker_planner.cpp.md) · [`action_planner_script.h`](action_planner_script.h.md) · [`action_script_base.h`](action_script_base.h.md) · [`action_planner_action_script.h`](action_planner_action_script.h.md) · [`stalker_planner_inline.h`](stalker_planner_inline.h.md)
**Used by** — [`ai_stalker.cpp`](ai/stalker/ai_stalker.cpp.md) · [`ai_stalker_debug.cpp`](ai/stalker/ai_stalker_debug.cpp.md) · [`ai_stalker_fire.cpp`](ai/stalker/ai_stalker_fire.cpp.md) · [`ai_stalker_script.cpp`](ai/stalker/ai_stalker_script.cpp.md) · [`script_game_object_use.cpp`](script_game_object_use.cpp.md) · [`stalker_base_action.cpp`](stalker_base_action.cpp.md) · [`stalker_combat_actions.cpp`](stalker_combat_actions.cpp.md) · [`stalker_combat_planner.cpp`](stalker_combat_planner.cpp.md) · [`stalker_low_cover_actions.cpp`](stalker_low_cover_actions.cpp.md) · [`stalker_low_cover_planner.cpp`](stalker_low_cover_planner.cpp.md) · [`stalker_planner.cpp`](stalker_planner.cpp.md) · [`stalker_planner_inline.h`](stalker_planner_inline.h.md)
**Tier floor** — T2: a planner instance over a small symbolic world state.

## Purpose

Declares the surface implemented in [`stalker_planner.cpp`](stalker_planner.cpp.md). It is
a planner whose operators are themselves planners, and which scripts may extend at
runtime — the script-extensible planner base is what it derives from, and that is a
frozen part of the modding surface.

## Exported units

- `setup(stalker)` — installs the evaluator set, the operator set and the goal.
- `update(time_delta)` — one planning-and-execution cycle.
- `affect_cover` (get and set) — a flag other systems read to decide whether the creature's
  current activity should influence squad cover bookkeeping.
- in diagnostic builds, three naming hooks that render an action identifier, a property
  identifier and the creature's own name as text for the planner log.
