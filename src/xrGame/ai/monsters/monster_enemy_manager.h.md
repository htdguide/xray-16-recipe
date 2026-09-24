# src/xrGame/ai/monsters/monster_enemy_manager.h

> Declares the enemy manager: the one enemy a creature is currently engaged with, its last known location, and the read of what that enemy is doing.

**Needs** — [`monster_enemy_manager.cpp`](monster_enemy_manager.cpp.md) · [`ai_monster_defs.h`](ai_monster_defs.h.md)
**Used by** — [`base_monster.h`](basemonster/base_monster.h.md) · [`melee_checker.cpp`](melee_checker.cpp.md) · [`monster_enemy_manager.cpp`](monster_enemy_manager.cpp.md) · [`monster_enemy_memory.cpp`](monster_enemy_memory.cpp.md)
**Tier floor** — T3: one selection, a snapshot, and a bit set recomputed each tick

## Purpose

Declares the surface implemented in [`monster_enemy_manager.cpp`](monster_enemy_manager.cpp.md).

## Exported units

- `bind`, `reinit` — attachment and reset.
- `update` — re-select, fold in sound, classify the threat, recompute the behaviour flags.
- `force_enemy` / `release_enemy` — pin a target, and return to the memory's choice.
- `script_enemy()` / `script_enemy(entity)` — the script-set target, which takes precedence
  over memory but not over a forced one.
- `enemy`, `enemy_position`, `enemy_vertex`, `enemy_time_last_seen`, `danger_type`, `flags` —
  the current selection and everything read off it.
- `sees_enemy_now`, `saw_enemy_recently`, `enemy_sees_me_now`, `sees_enemy_duration` — the
  perception questions.
- `enemy_count`, `remember`, `is_enemy`, `is_faced` — set size, admission, hostility test, and
  the shared "is A looking at B" predicate.
- `my_vertex_when_last_seen`, `enemy_vertex_when_last_seen` — the two navigation vertices
  recorded at the last actual sighting, used to resume a chase.
- `transfer_enemy` — copy a pack-mate's target into this creature's memory.
- `forget_entity`.
