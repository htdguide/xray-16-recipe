# src/xrGame/ai/monsters/monster_enemy_memory.h

> Declares the enemy memory: the hostile entities a creature currently knows about, each scored by a danger value.

**Needs** — [`monster_enemy_memory.cpp`](monster_enemy_memory.cpp.md) · [`ai_monster_defs.h`](ai_monster_defs.h.md)
**Used by** — [`base_monster.h`](basemonster/base_monster.h.md) · [`monster_enemy_manager.cpp`](monster_enemy_manager.cpp.md) · [`monster_enemy_memory.cpp`](monster_enemy_memory.cpp.md)
**Tier floor** — T3: a keyed collection with a scoring and eviction pass

## Purpose

Declares the surface implemented in [`monster_enemy_memory.cpp`](monster_enemy_memory.cpp.md).

## Exported units

- `bind` — attaches to a creature and sets how long a sighting is remembered.
- `update` — the per-tick step: admit, evict, re-score.
- `best_enemy` / `best_enemy_info` — the most dangerous known enemy, preferring those inside
  the creature's home area, and its recorded position, vertex and time.
- `count`, `entries` — size and read-only access to the whole set.
- `remember(enemy)` / `remember(enemy, position, vertex, time)` — admit from own perception, or
  from another creature's report.
- `clear`, `forget_entity`.
