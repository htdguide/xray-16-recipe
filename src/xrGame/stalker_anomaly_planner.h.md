# src/xrGame/stalker_anomaly_planner.h

> Declares the anomaly sub-planner — the branch of a stalker's brain that deals with hazardous ground.

**Needs** — [`stalker_anomaly_planner.cpp`](stalker_anomaly_planner.cpp.md) · [`action_planner_action_script.h`](action_planner_action_script.h.md) · [`ai/stalker/ai_stalker.h`](ai/stalker/ai_stalker.h.md)
**Used by** — [`stalker_anomaly_planner.cpp`](stalker_anomaly_planner.cpp.md) · [`stalker_planner.cpp`](stalker_planner.cpp.md)
**Tier floor** — T2: a planner that is also an operator in its parent's plan.

## Purpose

Declares the surface implemented in
[`stalker_anomaly_planner.cpp`](stalker_anomaly_planner.cpp.md). Its base is the
*planner-as-action* form: an object that presents preconditions and effects to the planner
above it while running a plan of its own below. That double identity is what makes the
stalker brain a tree, and it is also a modding surface — scripts may add evaluators and
operators to this planner at runtime.

## Exported units

- `setup(creature, parent_property_storage)` — bind to the creature, reset the shared
  proposition, and rebuild the evaluator and operator tables.
- `update()` — run one cycle and publish the result upward.
- `add_evaluators()` / `add_actions()` — the two halves of the rebuild, separated so a
  derived planner can replace one.
