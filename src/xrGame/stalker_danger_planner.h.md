# src/xrGame/stalker_danger_planner.h

> Declares the danger branch of a stalker's brain — the router that picks which kind of danger it is reacting to.

**Needs** — [`stalker_danger_planner.cpp`](stalker_danger_planner.cpp.md) · [`stalker_danger_planner_inline.h`](stalker_danger_planner_inline.h.md) · [`action_planner_action_script.h`](action_planner_action_script.h.md) · [`ai/stalker/ai_stalker.h`](ai/stalker/ai_stalker.h.md)
**Used by** — [`stalker_danger_planner.cpp`](stalker_danger_planner.cpp.md) · [`stalker_danger_planner_inline.h`](stalker_danger_planner_inline.h.md) · [`stalker_planner.cpp`](stalker_planner.cpp.md)
**Tier floor** — T2: a planner that is also an operator in its parent's plan.

## Purpose

Declares the surface implemented in
[`stalker_danger_planner.cpp`](stalker_danger_planner.cpp.md). Script-extensible, like
every planner in the brain.

## Exported units

- `setup(creature, parent_property_storage)` — rebuild the evaluator and operator tables.
- `initialize()` / `finalize()` — branch entry and exit, which do rather more than the
  usual nothing; see the implementation twin.
- `update()` — one cycle, plus two per-cycle reactions that belong to the whole branch.
- `add_evaluators()` / `add_actions()`.
