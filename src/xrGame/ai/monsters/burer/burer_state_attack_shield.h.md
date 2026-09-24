# src/xrGame/ai/monsters/burer/burer_state_attack_shield.h

> Declares the bullet shield as a timed behaviour state.

**Needs** — [`state.h`](../state.h.md) · [`burer_state_attack_shield_inline.h`](burer_state_attack_shield_inline.h.md)
**Used by** — [`burer_state_attack_inline.h`](burer_state_attack_inline.h.md) · [`burer_state_attack_shield_inline.h`](burer_state_attack_shield_inline.h.md)
**Tier floor** — T3: a timed state over a creature flag

## Purpose

Declares the surface implemented in [`burer_state_attack_shield_inline.h`](burer_state_attack_shield_inline.h.md). The shield's *effect* — swallowing damage and decals — lives on the creature in [`burer.cpp`](burer.cpp.md); this state owns only when it is up.

## `BurerShieldState`

A leaf state holding the tick the shield was raised, the next tick a keep-alive effect may be spawned, the length of the raise clip, and whether the shield has actually been raised yet.
