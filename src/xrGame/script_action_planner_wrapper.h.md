# src/xrGame/script_action_planner_wrapper.h

> Declares the adapter that lets a script define a creature's whole brain.

**Needs** — [`action_planner.h`](action_planner.h.md) · [`script_action_planner_wrapper_inline.h`](script_action_planner_wrapper_inline.h.md) · [`script_action_planner_wrapper.cpp`](script_action_planner_wrapper.cpp.md) · [Seam: Script binding layer](../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer)
**Used by** — [`action_planner_script.cpp`](action_planner_script.cpp.md) · [`script_action_planner_wrapper.cpp`](script_action_planner_wrapper.cpp.md)
**Tier floor** — T2: a call convention across the script boundary, once per decision cycle

## Purpose

Declares the surface implemented in
[`script_action_planner_wrapper.cpp`](script_action_planner_wrapper.cpp.md). A **planner** is
a creature's decision loop: it holds the operators and evaluators, re-solves for a plan every
update and executes the plan's first step (see
[`action_planner_inline.h`](action_planner_inline.h.md)). This adapter makes two of its points
overridable from a script class, so an entire brain — which operators exist, which goal is
pursued — can be authored in Lua.

Only two points are exposed, and the choice is the design: **a script may decide what the
brain is made of and when it thinks, but not how it solves.** The search itself stays in the
engine.

## Exported units

- `setup(object)` — bind the planner to its creature and install its parts. Paired with a
  base form that runs the inherited binding.
- `update` — one decision cycle. Paired with a base form that runs the inherited cycle.

**Notes** — this type has no constructor of its own and no state of its own; the inline
companion is empty. The adapter exists purely as a dispatch surface.
