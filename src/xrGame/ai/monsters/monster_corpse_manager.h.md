# src/xrGame/ai/monsters/monster_corpse_manager.h

> Declares the corpse manager: the one corpse a creature is currently interested in, either chosen from memory or forced by script.

**Needs** — [`monster_corpse_manager.cpp`](monster_corpse_manager.cpp.md) · [`ai_monster_defs.h`](ai_monster_defs.h.md)
**Used by** — [`base_monster.h`](basemonster/base_monster.h.md) · [`monster_corpse_manager.cpp`](monster_corpse_manager.cpp.md) · [`monster_state_eat_drag_inline.h`](states/monster_state_eat_drag_inline.h.md) · [`monster_state_eat_eat_inline.h`](states/monster_state_eat_eat_inline.h.md) · [`monster_state_eat_inline.h`](states/monster_state_eat_inline.h.md) · [`monster_state_rest_fun_inline.h`](states/monster_state_rest_fun_inline.h.md)
**Tier floor** — T3: one selected reference plus a snapshot of where it was

## Purpose

Declares the surface implemented in [`monster_corpse_manager.cpp`](monster_corpse_manager.cpp.md).

## Exported units

- `bind` — attaches to a creature.
- `update` — re-selects the current corpse, unless one is forced.
- `force_corpse` / `release_corpse` — script override and its removal.
- `corpse`, `corpse_position`, `corpse_vertex`, `corpse_time_last_seen` — the current
  selection and its snapshot.
- `reinit`, `forget_entity` — reset and destroyed-entity cleanup.
