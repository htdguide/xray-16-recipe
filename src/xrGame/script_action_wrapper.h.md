# src/xrGame/script_action_wrapper.h

> Declares the adapter that lets a script define a planner operator — an action with preconditions, effects and a cost.

**Needs** — [`action_base.h`](action_base.h.md) · [`script_action_wrapper_inline.h`](script_action_wrapper_inline.h.md) · [`script_action_wrapper.cpp`](script_action_wrapper.cpp.md) · [Seam: Script binding layer](../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer)
**Used by** — [`action_base_script.cpp`](action_base_script.cpp.md) · [`script_action_wrapper.cpp`](script_action_wrapper.cpp.md) · [`script_action_wrapper_inline.h`](script_action_wrapper_inline.h.md)
**Tier floor** — T2: a call convention across the script boundary, on the planner's inner loop

## Purpose

Declares the surface implemented in
[`script_action_wrapper.cpp`](script_action_wrapper.cpp.md). An **operator** is one action
the planner may schedule: it declares which world properties must hold before it can run,
which it establishes, and what it costs. This adapter makes all five of an operator's
overridable points reachable from a script class, so a modder can add a behaviour without
touching the engine.

Together with the evaluator adapter
([`script_property_evaluator_wrapper.h`](script_property_evaluator_wrapper.h.md)) it is what
makes the whole goal-and-plan layer scriptable — the pair is the reason the shipped games'
creature behaviour lives mostly in Lua.

## Exported units

Each appears twice, following the standard shape for script-overridable types in this
directory: one form dispatches into the script class, the other calls the inherited
behaviour so a script that overrode the method can still reach it without re-entering itself.

- construct from (game object, action name). The name is a diagnostic label.
- `setup(object, storage)` — bind the operator to its subject and to the shared world-state
  storage. Paired with a base form.
- `initialize` — the operator has just become the running action. Paired.
- `execute` — one update of the running action. Paired.
- `finalize` — the operator has stopped being the running action. Paired.
- `weight(from_state, to_state)` — the cost of this operator as a search edge between two
  world states. Paired.

**Notes** — the enter/run/leave trio is the same rhythm the rat states and the animation
actions use; see [`rat_state_base.h`](rat_state_base.h.md). Reproducing the rhythm matters
more than any individual part, because every script operator in the shipped game is written
against it.
