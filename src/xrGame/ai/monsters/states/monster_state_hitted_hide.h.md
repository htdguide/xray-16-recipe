# src/xrGame/ai/monsters/states/monster_state_hitted_hide.h

> Declares the break-away leaf, implemented in
> [`monster_state_hitted_hide_inline.h`](monster_state_hitted_hide_inline.h.md).

**Needs** — [`../state.h`](../state.h.md) · [`monster_state_hitted_hide_inline.h`](monster_state_hitted_hide_inline.h.md)
**Used by** — [`monster_state_hitted_hide_inline.h`](monster_state_hitted_hide_inline.h.md) · [`monster_state_hitted_inline.h`](monster_state_hitted_inline.h.md)
**Tier floor** — T3: a declaration

## Purpose

Names the leaf in which a creature that has been hit by an unseen attacker runs directly away from
the hit direction. Carries the two numbers that define it: the fifteen-unit distance that counts as
having broken away, and a minimum-duration guard.

## `CStateMonsterHittedHide`

- **enter** — prepare the path builder
- **execute** — run away from the last hit position, panicking
- **is_startable** — there is a recent hit and no enemy
- **is_finished** — fifteen units away, and past the minimum duration

Stateless beyond the base's entry timestamp. Contracts and the defect in the duration guard are in
the implementation twin.
