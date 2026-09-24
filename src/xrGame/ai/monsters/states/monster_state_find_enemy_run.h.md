# src/xrGame/ai/monsters/states/monster_state_find_enemy_run.h

> Declares the charge-to-last-known-position leaf, implemented in
> [`monster_state_find_enemy_run_inline.h`](monster_state_find_enemy_run_inline.h.md).

**Needs** — [`../state.h`](../state.h.md) · [`monster_state_find_enemy_run_inline.h`](monster_state_find_enemy_run_inline.h.md)
**Used by** — [`monster_state_find_enemy_inline.h`](monster_state_find_enemy_inline.h.md) · [`monster_state_find_enemy_run_inline.h`](monster_state_find_enemy_run_inline.h.md)
**Tier floor** — T3: a declaration

## Purpose

Names the first leaf of the search: a full-speed run to a point beyond where the enemy was last
seen. Its private state is that target — a position and a navigation vertex — resolved once on
entry and never revised.

## `CStateMonsterFindEnemyRun`

- **enter** — choose the overshoot target
- **execute** — run there using cover-biased pathing
- **is_finished** — standing on the target vertex with no path left
