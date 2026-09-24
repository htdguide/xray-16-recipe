# src/xrGame/stalker_search_planner.h

> Declares the lost-enemy search sub-planner.

**Needs** — [`stalker_search_planner.cpp`](stalker_search_planner.cpp.md) · [`action_planner_action_script.h`](action_planner_action_script.h.md)
**Used by** — [`stalker_combat_planner.cpp`](stalker_combat_planner.cpp.md) · [`stalker_search_planner.cpp`](stalker_search_planner.cpp.md)
**Tier floor** — T2: a planner instance.

## Purpose

Declares the surface implemented in
[`stalker_search_planner.cpp`](stalker_search_planner.cpp.md). It is a planner that is
itself an operator — the shape the whole stalker brain is built from — and one that scripts
may extend at runtime, which is part of the frozen modding surface.

## Exported units

- `setup(stalker, storage)` — installs the three evaluators and three operators.
- `initialize()` — releases the squad cover reservation and clears both progress flags in
  the parent's world state.
- `update()` / `finalize()` — delegation to the planner base.
- privately, the two table-filling steps `add_evaluators` and `add_actions`, split only for
  readability.
