# src/xrGame/script_property_evaluator_wrapper.h

> Declares the adapter that lets a script define a planner evaluator — one question about the world the planner can ask.

**Needs** — [`property_evaluator.h`](property_evaluator.h.md) · [`script_game_object.h`](script_game_object.h.md) · [`script_property_evaluator_wrapper.cpp`](script_property_evaluator_wrapper.cpp.md) · [Seam: Script binding layer](../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer)
**Used by** — [`property_evaluator_script.cpp`](property_evaluator_script.cpp.md) · [`script_property_evaluator_wrapper.cpp`](script_property_evaluator_wrapper.cpp.md) · [`script_property_evaluator_wrapper_inline.h`](script_property_evaluator_wrapper_inline.h.md)
**Tier floor** — T2: a call convention across the script boundary

## Purpose

Declares the surface implemented in
[`script_property_evaluator_wrapper.cpp`](script_property_evaluator_wrapper.cpp.md). An
**evaluator** answers one yes-or-no question about the current world state, and the
planner's operators express their preconditions and effects over those answers. This
adapter makes the evaluator's two overridable points reachable from a script class, so a
modder can add a new question without touching the engine.

It also fixes the evaluator's subject: a script evaluator is always asked about a
[game object](../../GLOSSARY.md), never about an arbitrary owner. The engine's evaluator is
generic over its subject; this specialization is what the script layer sees.

## Exported units

- construct from (game object, evaluator name). The name is kept for diagnostics only.
- `setup(object, storage)` — dispatched to the script's own `setup`; binds the evaluator to
  its subject and to the shared property storage the planner reads answers out of.
- `setup_base(evaluator, object, storage)` — the inherited behaviour, callable from a
  script that overrode `setup` and still wants it.
- `evaluate` — dispatched to the script's own `evaluate`; returns the answer.
- `evaluate_base(evaluator)` — the inherited answer, for the same reason.

**Notes**

The paired "call the override" and "call the base" methods are the standard shape for every
script-overridable type in this directory; see
[`script_effector_wrapper.cpp`](script_effector_wrapper.cpp.md) for the same two functions
on a different type. The base-calling half exists because a script method that overrides
`evaluate` would otherwise have no way to reach the inherited one without re-entering
itself.
