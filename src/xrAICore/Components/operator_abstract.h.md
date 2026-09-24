# src/xrAICore/Components/operator_abstract.h

> Declares a planner operator — preconditions, effects, cost — whose algorithms live in [`operator_abstract_inline.h`](operator_abstract_inline.h.md).

**Needs** — [`condition_state.h`](condition_state.h.md) · [`operator_abstract_inline.h`](operator_abstract_inline.h.md)
**Used by** — [`operator_abstract_inline.h`](operator_abstract_inline.h.md) · [`problem_solver.h`](problem_solver.h.md) · [`script_world_property_script.cpp`](script_world_property_script.cpp.md) · [`graph_engine_space.h`](../Navigation/graph_engine_space.h.md) · [`action_base.h`](../../xrGame/action_base.h.md)
**Tier floor** — T2: ordered-merge state algebra over abstract property identifiers.

## Purpose

Declares the surface implemented in
[`operator_abstract_inline.h`](operator_abstract_inline.h.md). An operator is one action the
planner may put in a plan: a partial world state it requires, a partial world state it
produces, and a cost. The game layer derives concrete actions from it; nothing here knows
what an action *does*, only what it needs and what it changes.

## Exported units

- `conditions()` / `effects()` — the two partial world states
- `add_condition` / `remove_condition`, `add_effect` / `remove_effect` — edit them, and in
  doing so invalidate both the cached cost and the owner's current plan
- `Load(section)` — read this operator's tuning from a configuration section; a no-op here,
  overridden by concrete actions
- `setup(actuality_flag)` — bind the flag that this operator clears when it is edited
- `min_weight()` — the cached lower bound on this operator's cost
- `weight(from, to)` — the cost of this operator between two states; defaults to `min_weight()`
  and is meant to be overridden with a data-driven cost
- `applicable(vertex, start, preconditions, solver)` — forward-search precondition test
- `apply(vertex, effects, result, start, solver)` — forward-search successor state
- `applicable_reverse(effects, preconditions, goal)` — backward-search applicability test
- `apply_reverse(goal, effects, result, preconditions)` — backward-search regressed goal
- `apply(state, other, result)` — the plain merge of two partial states
