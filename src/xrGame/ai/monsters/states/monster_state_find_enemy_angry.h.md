# src/xrGame/ai/monsters/states/monster_state_find_enemy_angry.h

> Declares the four-second threat display, implemented in
> [`monster_state_find_enemy_angry_inline.h`](monster_state_find_enemy_angry_inline.h.md).

**Needs** — [`../state.h`](../state.h.md) · [`monster_state_find_enemy_angry_inline.h`](monster_state_find_enemy_angry_inline.h.md)
**Used by** — [`monster_state_find_enemy_angry_inline.h`](monster_state_find_enemy_angry_inline.h.md) · [`monster_state_find_enemy_inline.h`](monster_state_find_enemy_inline.h.md)
**Tier floor** — T3: a declaration

## Purpose

Names the third leaf of the lost-contact search: stand still in a threatening posture, snarling, for
a fixed four seconds. Stateless beyond the base's entry timestamp.

## `CStateMonsterFindEnemyAngry`

- **execute** — stand idle with the threat animation flag and the aggressive voice
- **is_finished** — four seconds after entry
