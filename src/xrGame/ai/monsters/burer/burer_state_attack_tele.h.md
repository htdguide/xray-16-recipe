# src/xrGame/ai/monsters/burer/burer_state_attack_tele.h

> Declares the telekinetic attack: find loose objects, lift them, throw them one at a time — and catch live grenades on the way past.

**Needs** — [`state.h`](../state.h.md) · [`telekinesis.h`](../telekinesis.h.md) · [`burer_state_attack_tele_inline.h`](burer_state_attack_tele_inline.h.md)
**Used by** — [`burer_state_attack_inline.h`](burer_state_attack_inline.h.md) · [`burer_state_attack_tele_inline.h`](burer_state_attack_tele_inline.h.md)
**Tier floor** — T2: proximity queries and mass filtering against the physics world

## Purpose

Declares the surface implemented in [`burer_state_attack_tele_inline.h`](burer_state_attack_tele_inline.h.md).

## `BurerAttackTeleState`

A leaf state holding the candidate list, the object selected for the next throw, a scratch buffer for proximity queries, the phase marker (started, holding, fire, wait for the throw clip, done), the time the lift began, the last grenade sweep time, the clip-end and overall deadline ticks, and the health the creature had at entry.

```text
max_time_without_a_throw = 6000 ms   # fixed in code; see the implementation
```
