# src/xrGame/ai/monsters/states/monster_state_find_enemy.h

> Declares the lost-contact search composite, implemented in
> [`monster_state_find_enemy_inline.h`](monster_state_find_enemy_inline.h.md).

**Needs** — [`../state.h`](../state.h.md) · [`monster_state_find_enemy_inline.h`](monster_state_find_enemy_inline.h.md)
**Used by** — [`group_state_attack_inline.h`](../group_states/group_state_attack_inline.h.md) · [`monster_state_attack_inline.h`](monster_state_attack_inline.h.md) · [`monster_state_find_enemy_inline.h`](monster_state_find_enemy_inline.h.md)
**Tier floor** — T3: a declaration

## Purpose

Names the behaviour a creature runs when it had an enemy and lost sight of it: charge the last
known position, cast about, threaten, then mill around. Selected from inside the attack behaviour,
not from the top-level dispatch.

## `CStateMonsterFindEnemy`

- **construct** — register the four search leaves
- **reselect** — advance one step along the fixed sequence

It owns no state of its own beyond the base's record of which leaf is current. Contracts are in the
implementation twin.
