# src/xrGame/agent_manager_planner.h

> Declares the squad planner as the generic planner specialized to the squad manager.

**Needs** — [`agent_manager_planner.cpp`](agent_manager_planner.cpp.md) · [`action_planner.h`](action_planner.h.md)
**Used by** — [`agent_manager.cpp`](agent_manager.cpp.md) · [`agent_manager_planner.cpp`](agent_manager_planner.cpp.md)
**Tier floor** — T3: a declaration.

## Purpose

Declares the surface implemented in
[`agent_manager_planner.cpp`](agent_manager_planner.cpp.md). The squad planner is the
engine's generic goal/plan search with the squad manager as the object every evaluator and
operator is handed.

Exported units:

- **`setup`** — bind to a squad manager and install the problem.
- **`add_evaluators`** / **`add_actions`** — the two halves of that installation, split so
  a derived planner could replace one.
- **`remove_links`** — no-op; see the cpp twin.
