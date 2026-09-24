# src/xrGame/ai/monsters/states/monster_state_hitted_moveout.h

> Declares the stalk-back leaf, implemented in
> [`monster_state_hitted_moveout_inline.h`](monster_state_hitted_moveout_inline.h.md).

**Needs** — [`../state.h`](../state.h.md) · [`monster_state_hitted_moveout_inline.h`](monster_state_hitted_moveout_inline.h.md) · [`../../../detail_path_manager.h`](../../../detail_path_manager.h.md)
**Used by** — [`monster_state_hitted_inline.h`](monster_state_hitted_inline.h.md) · [`monster_state_hitted_moveout_inline.h`](monster_state_hitted_moveout_inline.h.md)
**Tier floor** — T3: a declaration

## Purpose

Names the leaf in which a creature that has broken away from an unseen shooter creeps back toward
the hit direction, hopping between covered spots. Carries two numbers: the distance that counts as
having finished a leg of the route, and the distance at which the creature is close enough to the
hit point to stop.

Its private state is the current leg's destination — a position and a navigation vertex, with the
vertex absent meaning "no cover was found, head straight for the hit point".

## `CStateMonsterHittedMoveOut`

- **enter** — choose the first covered destination and prepare the path builder
- **execute** — re-choose when a leg finishes; walk or creep depending on range
- **is_finished** — hit again, or within three units of the hit point
