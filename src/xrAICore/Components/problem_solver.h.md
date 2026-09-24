# src/xrAICore/Components/problem_solver.h

> Declares the planner — an operator set, a property-evaluator set, a goal, and the graph face it presents to the search engine — implemented in [`problem_solver_inline.h`](problem_solver_inline.h.md).

**Needs** — [`problem_solver_inline.h`](problem_solver_inline.h.md) · [`operator_abstract.h`](operator_abstract.h.md) · [`condition_state.h`](condition_state.h.md) · [`xrCore/Containers/AssociativeVector.hpp`](../../xrCore/Containers/AssociativeVector.hpp.md) · [`Common/object_broker.h`](../../Common/object_broker.h.md)
**Used by** — [`problem_solver_inline.h`](problem_solver_inline.h.md) · [`path_manager_solver.h`](../Navigation/PathManagers/path_manager_solver.h.md) · [`graph_engine.h`](../Navigation/graph_engine.h.md) · [`action_planner.h`](../../xrGame/action_planner.h.md)
**Tier floor** — T2: a search over symbolic states; the only hard budget is a visited-state cap.

## Purpose

Declares the surface implemented in
[`problem_solver_inline.h`](problem_solver_inline.h.md). The type is parameterised on the
property type, the state type, the operator type, the property-evaluator type, the operator
identifier type, and one compile-time choice: whether the search runs forward from the world
to the goal, or backward from the goal to the world. The default is forward and the game layer
takes the default.

Its unusual shape is worth stating once: the planner is *itself the graph* the search engine
walks. A state is a vertex, an operator is an edge, and the routines grouped as the graph
interface below are what the search engine calls. Nothing else in the codebase implements a
graph this way — every other graph in this chapter is a real stored structure.

## Exported units

Lifecycle:

- `init()` / `setup()` — reset the goal, the measured world, the plan and the staleness flags
- `clear()` — release every operator and evaluator; the planner owns them
- `actual()` — is the standing plan still valid

Graph face, called by the search engine:

- `begin(state)` — the outgoing edges of a state: *every* operator, unfiltered
- `value(state, edge, reverse)` — the state on the far side of an operator, and the side effect
  of recording whether the operator was applicable at all
- `is_accessible(state)` — whether the last `value` produced a real successor
- `get_edge_weight(from, to, edge)` — that operator's cost, checked against its lower bound
- `is_goal_reached(state)` — termination test, direction-dependent
- `estimate_edge_weight(state)` — the heuristic, direction-dependent

Operators, evaluators, goal:

- `add_operator(id, operator)` / `remove_operator(id)` / `get_operator(id)` / `operators()`
- `add_evaluator(property_id, evaluator)` / `remove_evaluator(property_id)` / `evaluator(id)` / `evaluators()`
- `evaluate_condition(...)` — measure one property of the world and cache it
- `set_target_state(state)` / `target_state()` / `current_state()`

Solving:

- `solve()` — re-plan if and only if the standing plan is stale
- `solution()` — the plan, as a sequence of operator identifiers

**Notes** — the template parameter names in the declaration and in the implementation file
disagree: positions two and three are called *state, operator* in one and *operator, state* in
the other. Nothing binds to those names except a macro, so it compiles and behaves correctly,
but a reader tracing types across the two files will be misled. A rebuild should not reproduce
the parameterisation at all — one planner type with one state type is enough.
