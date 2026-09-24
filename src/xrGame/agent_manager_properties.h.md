# src/xrGame/agent_manager_properties.h

> Declares the three squad evaluators and the squad-bound aliases of the generic evaluator kinds.

**Needs** — [`agent_manager_properties.cpp`](agent_manager_properties.cpp.md) · [`property_evaluator_const.h`](property_evaluator_const.h.md) · [`property_evaluator_member.h`](property_evaluator_member.h.md) · [`agent_manager_properties_inline.h`](agent_manager_properties_inline.h.md)
**Used by** — [`agent_manager_planner.cpp`](agent_manager_planner.cpp.md) · [`agent_manager_properties.cpp`](agent_manager_properties.cpp.md) · [`agent_manager_properties_inline.h`](agent_manager_properties_inline.h.md)
**Tier floor** — T3: declarations.

## Purpose

Declares the surface implemented in
[`agent_manager_properties.cpp`](agent_manager_properties.cpp.md), plus three names the
planner wiring uses:

- **`AgentManagerPropertyEvaluator`** — the generic evaluator bound to the squad manager.
- **`AgentManagerPropertyEvaluatorConst`** — one that ignores the world and returns a fixed
  answer; the squad's `Orders` property is one of these, always false.
- **`AgentManagerPropertyEvaluatorMember`** — the member-scoped variant.

Exported units: **`EvaluatorItem`**, **`EvaluatorEnemy`**, **`EvaluatorDanger`**, each
constructed with the squad manager and a name for plan traces.
