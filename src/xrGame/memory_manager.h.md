# src/xrGame/memory_manager.h

> Declares a creature's whole memory: the three senses, and the three derived judgements built on them.

**Needs** — [`memory_manager.cpp`](memory_manager.cpp.md) · [`memory_manager_inline.h`](memory_manager_inline.h.md) · [`memory_space.h`](memory_space.h.md) · [`enemy_manager.h`](enemy_manager.h.md) · [`item_manager.h`](item_manager.h.md) · [`danger_manager.h`](danger_manager.h.md) · [`visual_memory_manager.h`](visual_memory_manager.h.md) · [`sound_memory_manager.h`](sound_memory_manager.h.md) · [`hit_memory_manager.h`](hit_memory_manager.h.md)
**Used by** — [`CustomMonster.cpp`](CustomMonster.cpp.md) · [`agent_corpse_manager.cpp`](agent_corpse_manager.cpp.md) · [`agent_enemy_manager.cpp`](agent_enemy_manager.cpp.md) · [`agent_explosive_manager.cpp`](agent_explosive_manager.cpp.md) · [`agent_manager_properties.cpp`](agent_manager_properties.cpp.md) · [`ai_monsters_misc.cpp`](ai/ai_monsters_misc.cpp.md) · [`ai_monster_squad.cpp`](ai/monsters/ai_monster_squad.cpp.md) · [`monster_corpse_memory.cpp`](ai/monsters/monster_corpse_memory.cpp.md) · [`monster_enemy_manager.cpp`](ai/monsters/monster_enemy_manager.cpp.md) · [`monster_enemy_memory.cpp`](ai/monsters/monster_enemy_memory.cpp.md) · [`ai_rat_feel.cpp`](ai/monsters/rats/ai_rat_feel.cpp.md) · [`ai_rat_fire.cpp`](ai/monsters/rats/ai_rat_fire.cpp.md) · [`rat_state_activation.cpp`](ai/monsters/rats/rat_state_activation.cpp.md) · [`rat_state_switch.cpp`](ai/monsters/rats/rat_state_switch.cpp.md) · _and 41 more_
**Tier floor** — T3: a declaration

## Purpose

Declares `CMemoryManager`. See [`memory_manager.cpp`](memory_manager.cpp.md).

Exported units:

- `CMemoryManager` — six owned sub-managers in two layers, plus the owning creature and, when
  it is a stalker, a second reference to it in its stalker form.
- `Load` · `reinit` · `reload` — the three configuration and lifecycle passes, fanned out.
- `update` — the per-frame cycle, in a load-bearing order.
- `remove_links` · `on_restrictions_change` · `on_requested_spawn` — the notifications.
- `enable` — mute or unmute one object across all three senses.
- `memory` · `memory_time` · `memory_position` — the merged answer about one object.
- `make_object_visible_somewhen` — inject a memory of having seen something.
- `fill_enemies` — enumerate remembered objects that qualify as enemies.
- `visual` · `sound` · `hit` · `enemy` · `item` · `danger` · `object` · `stalker` — the
  sub-manager accessors, in
  [`memory_manager_inline.h`](memory_manager_inline.h.md).
- `save` · `load` — persist the three senses and the danger record.
