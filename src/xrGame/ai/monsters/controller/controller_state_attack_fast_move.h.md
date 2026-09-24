# src/xrGame/ai/monsters/controller/controller_state_attack_fast_move.h

> Declares a sprint-between-cover state for the controller. Nothing registers it.

**Needs** — [`state.h`](../state.h.md) · [`controller_state_attack_fast_move_inline.h`](controller_state_attack_fast_move_inline.h.md)
**Used by** — [`controller_state_attack_fast_move_inline.h`](controller_state_attack_fast_move_inline.h.md)
**Tier floor** — T3: a declaration

## Purpose

Declares the surface implemented in [`controller_state_attack_fast_move_inline.h`](controller_state_attack_fast_move_inline.h.md). No state manager or composite in the shipped build constructs it; the controller's retreat is handled by [`controller_state_attack_hide_inline.h`](controller_state_attack_hide_inline.h.md) instead.

## `ControllerFastMoveState`

A leaf state with no fields, overriding entry, update and both exits.
