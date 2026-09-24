# src/xrGame/ai/monsters/burer/burer_state_attack_gravi.h

> Declares the gravity attack: charge, hold for longer the further the enemy is, release a wave.

**Needs** — [`state.h`](../state.h.md) · [`burer_state_attack_gravi_inline.h`](burer_state_attack_gravi_inline.h.md)
**Used by** — [`burer_state_attack_gravi_inline.h`](burer_state_attack_gravi_inline.h.md) · [`burer_state_attack_inline.h`](burer_state_attack_inline.h.md)
**Tier floor** — T2: a timed phase machine driving a physical ability

## Purpose

Declares the surface implemented in [`burer_state_attack_gravi_inline.h`](burer_state_attack_gravi_inline.h.md).

## `BurerAttackGraviState`

A leaf state holding a five-step phase marker (started, charging, fire, wait for the clip to end, done), the time the charge began, the tick at which the next gravity attack becomes legal, and the tick at which the current clip ends.
