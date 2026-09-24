# src/xrGame/action_planner.h

> Declares the brain: the component that holds a creature's evaluators and actions, searches for a plan from the world as it is to the world as it wants it, and runs the first step. Behaviour is in [`action_planner_inline.h`](action_planner_inline.h.md).

**Needs** — [`action_base.h`](action_base.h.md) · [`property_evaluator.h`](property_evaluator.h.md) · [`property_storage.h`](property_storage.h.md) · [`ai_debug.h`](ai_debug.h.md) · [`action_planner_inline.h`](action_planner_inline.h.md) · [`xrAICore/Components/problem_solver.h`](../xrAICore/Components/problem_solver.h.md) · [`xrAICore/Navigation/graph_engine.h`](../xrAICore/Navigation/graph_engine.h.md) · [Seam: Script binding layer](../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer)
**Used by** — [`action_planner_action.h`](action_planner_action.h.md) · [`action_planner_action_inline.h`](action_planner_action_inline.h.md) · [`action_planner_inline.h`](action_planner_inline.h.md) · [`action_planner_script.cpp`](action_planner_script.cpp.md) · [`action_planner_script.h`](action_planner_script.h.md) · [`action_planner_script_inline.h`](action_planner_script_inline.h.md) · [`agent_manager_planner.h`](agent_manager_planner.h.md) · [`object_handler_planner.h`](object_handler_planner.h.md) · [`script_action_planner_wrapper.h`](script_action_planner_wrapper.h.md) · [`script_game_object_script2.cpp`](script_game_object_script2.cpp.md) · [`UIMapWndActions.cpp`](ui/UIMapWndActions.cpp.md) · [`UIMapWndActions.h`](ui/UIMapWndActions.h.md)
**Tier floor** — T3: a declaration; the parameterization is a language convenience

## Purpose

Declares the game layer's specialization of the AI core's problem solver: world properties
are boolean-valued named conditions, world states are sets of them, operators are actions
and conditions are answered by evaluators. Everything this adds over the solver is
*lifecycle* — the solver finds a plan, this decides what to do with it.

It is parameterized over the acting object, over whether the search runs forward from the
current state or backward from the goal, and over the action and evaluator types, so that
one implementation serves stalkers, monsters, the actor's own auxiliary brains and
script-defined brains. The script-facing instantiation is fixed to the game object facade
and named here.

Substance is in [`action_planner_inline.h`](action_planner_inline.h.md).

Exported units:

- `setup` — bind the acting object and reset to the uninitialized state.
- `update` — the per-decision entry point: solve, switch action if the plan's head
  changed, execute.
- `finalize` — abandon the current action.
- `add_operator` / `remove_operator`, `add_evaluator` / `remove_evaluator` — build the
  brain; adding also binds the shared world-state storage into the new part.
- `add_condition` / `add_effect` — declare an action's precondition or effect through the
  planner, so the "not while solving" rule is enforced in one place.
- `action`, `evaluator`, `current_action`, `current_action_id`, `initialized` — queries.
- `save` / `load` — serialize every evaluator, every action, and the world-state storage.
- `object` — the acting object.
- A registration helper type carrying the script export; see
  [`action_planner_script.cpp`](action_planner_script.cpp.md).

**Notes**

- The shared world-state storage is a *member*, not a pointer: the planner owns it and
  hands a reference to every action and evaluator it is given. That ownership is the
  reason a creature's evaluators can cache into the same place its actions read from.
- The debug-only tracing members change the type's layout between build configurations,
  the same hazard noted in
  [`action_management_config.h`](action_management_config.h.md).
