# src/xrGame/ai/monsters/states/monster_state_rest_fun.h

> Declares the play-with-a-corpse leaf, implemented in
> [`monster_state_rest_fun_inline.h`](monster_state_rest_fun_inline.h.md). **Dead behaviour**: it is
> registered in the solitary peacetime composite and selected by nothing, anywhere.

**Needs** — [`../state.h`](../state.h.md) · [`monster_state_rest_fun_inline.h`](monster_state_rest_fun_inline.h.md) · [`../../../ai_debug.h`](../../../ai_debug.h.md)
**Used by** — [`monster_state_rest_fun_inline.h`](monster_state_rest_fun_inline.h.md) · [`monster_state_rest_inline.h`](monster_state_rest_inline.h.md)
**Tier floor** — T3: a declaration

## Purpose

Names the leaf in which an idle creature would run at a corpse and bat it around. It is fully
implemented, registered under the identifier for "playing" in
[`monster_state_rest_inline.h`](monster_state_rest_inline.h.md), and that identifier appears in no
selection anywhere in the source — not in the solitary peacetime cascade, not in the pack variant,
not in any species-specific tree. The timestamp field the peacetime composite declares for gating
it is written once and never read.

It carries three compiled-in numbers: the impulse applied to the body, the minimum gap between
swipes, and the eight seconds the leaf would run for.

## `CStateMonsterRestFun`

- **enter** — reset the swipe clock
- **execute** — run to just past the corpse and shove it when close enough
- **is_startable** — there is a corpse and it lies inside the territory
- **is_finished** — the corpse is gone, or eight seconds have passed

Contracts are in the implementation twin, which describes it as written rather than as running.
