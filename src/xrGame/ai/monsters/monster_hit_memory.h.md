# src/xrGame/ai/monsters/monster_hit_memory.h

> Declares the hit memory: who has recently hurt this creature, and from which side.

**Needs** — [`monster_hit_memory.cpp`](monster_hit_memory.cpp.md) · [`ai_monster_defs.h`](ai_monster_defs.h.md)
**Used by** — [`base_monster.h`](basemonster/base_monster.h.md) · [`monster_enemy_memory.cpp`](monster_enemy_memory.cpp.md) · [`monster_hit_memory.cpp`](monster_hit_memory.cpp.md)
**Tier floor** — T3: a small list with a time-based eviction pass

## Purpose

Declares the surface implemented in [`monster_hit_memory.cpp`](monster_hit_memory.cpp.md).

## Exported units

- `bind` — attach to a creature and set how long a hit is remembered.
- `update` — evict stale hits.
- `has_hits`, `was_hit_by`, `hit_count` — the presence questions.
- `record_hit(source, side)` — admit one hit.
- `last_hit_direction`, `last_hit_time`, `last_hit_source`, `last_hit_position` — facts about
  the newest remembered hit.
- `clear`, `forget_entity`.
