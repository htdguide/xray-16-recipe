# src/xrGame/ai/monsters/states/monster_state_find_enemy_walk.h

> Declares the absorbing final leaf of the search, implemented in
> [`monster_state_find_enemy_walk_inline.h`](monster_state_find_enemy_walk_inline.h.md).

**Needs** — [`../state.h`](../state.h.md) · [`monster_state_find_enemy_walk_inline.h`](monster_state_find_enemy_walk_inline.h.md)
**Used by** — [`monster_state_find_enemy_inline.h`](monster_state_find_enemy_inline.h.md) · [`monster_state_find_enemy_walk_inline.h`](monster_state_find_enemy_walk_inline.h.md)
**Tier floor** — T3: a declaration

## Purpose

Names the leaf the lost-contact search settles into and never leaves under its own power. Its
completion test answers "no" unconditionally, which is what makes it absorbing.

## `CStateMonsterFindEnemyWalkAround`

- **execute** — stand idle with the aggressive voice
- **is_finished** — never
