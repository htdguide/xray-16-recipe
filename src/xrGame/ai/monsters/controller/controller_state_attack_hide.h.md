# src/xrGame/ai/monsters/controller/controller_state_attack_hide.h

> Declares the controller's run-to-cover state, implemented in
> [`controller_state_attack_hide_inline.h`](controller_state_attack_hide_inline.h.md).

**Needs** — [`../state.h`](../state.h.md) · [`controller_state_attack_hide_inline.h`](controller_state_attack_hide_inline.h.md)
**Used by** — [`controller_state_attack_hide_inline.h`](controller_state_attack_hide_inline.h.md) · [`controller_state_attack_inline.h`](controller_state_attack_inline.h.md) · [`controller_state_manager.cpp`](controller_state_manager.cpp.md)
**Tier floor** — T3: a declaration

## Purpose

Names the surface of one leaf state of the controller's brain, so that the state manager can
register it without seeing the implementation. The split between this file and its
implementation is an artefact of how C++ separates a template's declaration from its body; a
rebuild has one unit.

## `CStateControlHide`

The state's exported units, all of them parts of the standard leaf-state contract:

- **enter** — pick a cover point and prime the path builder
- **execute** — drive movement, animation, sound and head-look each tick
- **leave** (clean and forced) — restore the creature's mental state
- **is_finished** — arrival predicate
- **may_start** — unconditionally yes

Its private state is a chosen target (world position plus navigation vertex), a flag saying this
activation began far enough away to warrant a sprint, and a finish stamp. The contracts are in
the implementation twin.
