# src/xrGame/ai/monsters/states/monster_state_rest_idle.h

> Declares the standing-around composite, implemented in
> [`monster_state_rest_idle_inline.h`](monster_state_rest_idle_inline.h.md).

**Needs** — [`../state.h`](../state.h.md) · [`monster_state_rest_idle_inline.h`](monster_state_rest_idle_inline.h.md)
**Used by** — [`monster_state_rest_idle_inline.h`](monster_state_rest_idle_inline.h.md) · [`monster_state_rest_inline.h`](monster_state_rest_inline.h.md)
**Tier floor** — T3: a declaration

## Purpose

Names what a creature does with the sixty-second idle window the peacetime cascade gives it: find a
covered spot, walk to it, turn to watch the open ground, and settle. This is the single most
frequently running behaviour in the world.

Its private state is the claimed cover vertex, chosen once on entry.

## `CStateMonsterRestIdle`

- **enter** — find and claim a covered spot
- **leave** (clean and forced) — release the claim
- **reselect** — walk, then look, then settle
- **setup** — fill each leaf's parameters
