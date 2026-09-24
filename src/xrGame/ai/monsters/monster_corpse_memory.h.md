# src/xrGame/ai/monsters/monster_corpse_memory.h

> Declares the corpse memory: the set of dead bodies a creature has seen recently and still considers edible.

**Needs** — [`monster_corpse_memory.cpp`](monster_corpse_memory.cpp.md) · [`ai_monster_defs.h`](ai_monster_defs.h.md)
**Used by** — [`base_monster.h`](basemonster/base_monster.h.md) · [`monster_corpse_manager.cpp`](monster_corpse_manager.cpp.md) · [`monster_corpse_memory.cpp`](monster_corpse_memory.cpp.md)
**Tier floor** — T3: a keyed collection with a time-based eviction pass

## Purpose

Declares the surface implemented in [`monster_corpse_memory.cpp`](monster_corpse_memory.cpp.md).

## Exported units

- `bind` — attaches to a creature and sets how long a sighting is remembered.
- `update` — the per-tick step: admit newly seen corpses, evict stale ones.
- `best_corpse` / `best_corpse_info` — the nearest remembered corpse, and its recorded
  position, navigation vertex and sighting time.
- `count` — how many corpses are remembered.
- `remember` — force a corpse into memory without having seen it.
- `is_remembered` — whether a given corpse is in the set.
- `clear`, `forget_entity` — wholesale and single-entry removal.
