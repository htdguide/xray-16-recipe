# src/xrGame/ai/monsters/burer/burer_state_attack_melee.h

> Declares a close-quarters attack the burer's attack tree registers and never selects.

**Needs** — [`monster_state_attack.h`](../states/monster_state_attack.h.md) · [`state.h`](../state.h.md) · [`burer_state_attack_melee_inline.h`](burer_state_attack_melee_inline.h.md)
**Used by** — [`burer_state_attack_inline.h`](burer_state_attack_inline.h.md) · [`burer_state_attack_melee_inline.h`](burer_state_attack_melee_inline.h.md)
**Tier floor** — T3: two distance predicates over a shared state

## Purpose

Declares the surface implemented in [`burer_state_attack_melee_inline.h`](burer_state_attack_melee_inline.h.md).

## `BurerAttackMeleeState`

Derives from the shared creature attack state and overrides only its two predicates. Adds no data.
