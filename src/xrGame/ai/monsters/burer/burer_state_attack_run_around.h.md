# src/xrGame/ai/monsters/burer/burer_state_attack_run_around.h

> Declares the burer's repositioning move: pick a spot, run to it, keep facing the enemy on arrival.

**Needs** — [`state.h`](../state.h.md) · [`burer_state_attack_run_around_inline.h`](burer_state_attack_run_around_inline.h.md)
**Used by** — [`burer_state_attack_inline.h`](burer_state_attack_inline.h.md) · [`burer_state_attack_run_around_inline.h`](burer_state_attack_run_around_inline.h.md)
**Tier floor** — T3: target selection and a movement command

## Purpose

Declares the surface implemented in [`burer_state_attack_run_around_inline.h`](burer_state_attack_run_around_inline.h.md).

## `BurerAttackRunAroundState`

A leaf state holding the chosen destination, the time the run began, and the facing the creature should have on arrival.
