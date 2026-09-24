# src/xrGame/script_action_planner_action_wrapper.h

> Declares the adapter for a script-defined operator that is itself a planner — one step of a plan that expands into a whole sub-plan.

**Needs** — [`action_planner_action.h`](action_planner_action.h.md) · [`script_game_object.h`](script_game_object.h.md) · [`script_action_planner_action_wrapper_inline.h`](script_action_planner_action_wrapper_inline.h.md) · [`script_action_planner_action_wrapper.cpp`](script_action_planner_action_wrapper.cpp.md) · [Seam: Script binding layer](../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer)
**Used by** — [`action_planner_action_script.cpp`](action_planner_action_script.cpp.md) · [`script_action_planner_action_wrapper.cpp`](script_action_planner_action_wrapper.cpp.md) · [`script_action_planner_action_wrapper_inline.h`](script_action_planner_action_wrapper_inline.h.md)
**Tier floor** — T2: a call convention across the script boundary, on a nested planner

## Purpose

Declares the surface implemented in
[`script_action_planner_action_wrapper.cpp`](script_action_planner_action_wrapper.cpp.md).

The idea it exists for is **hierarchical planning**: an operator in the outer plan can be a
planner in its own right, with its own goal, its own operators and its own evaluators. The
outer planner sees a single action with preconditions, effects and a cost; when that action
runs, it runs its own search and executes its own first step. This is what keeps a creature's
top-level plan a handful of steps long while its behaviour is arbitrarily deep.

This adapter is that composite made scriptable. Its exported surface is identical to the
plain operator adapter's ([`script_action_wrapper.h`](script_action_wrapper.h.md)) — the same
five overridable points, each paired with a base-calling form — because *that is the point*:
a sub-planner is indistinguishable from an action to its parent.

## Exported units

- construct from (game object, action name).
- `setup(object, storage)` and its base form.
- `initialize`, `execute`, `finalize`, each with its base form.
- `weight(from_state, to_state)` and its base form.

**Notes** — the enter/run/leave trio here brackets the *sub-planner's* life, not one action's:
initializing means the sub-plan starts being solved, finalizing means its running action gets
its closing call. A script overriding these without calling the base forms will leave the
nested planner unsolved and its inner actions unbracketed.
