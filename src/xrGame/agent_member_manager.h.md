# src/xrGame/agent_member_manager.h

> Declares the squad roster and its mask vocabulary.

**Needs** — [`agent_member_manager.cpp`](agent_member_manager.cpp.md) · [`member_order.h`](member_order.h.md) · [`memory_space.h`](memory_space.h.md) · [`agent_member_manager_inline.h`](agent_member_manager_inline.h.md)
**Used by** — [`agent_corpse_manager.cpp`](agent_corpse_manager.cpp.md) · [`agent_enemy_manager.cpp`](agent_enemy_manager.cpp.md) · [`agent_explosive_manager.cpp`](agent_explosive_manager.cpp.md) · [`agent_location_manager.cpp`](agent_location_manager.cpp.md) · [`agent_manager.cpp`](agent_manager.cpp.md) · [`agent_manager_actions.cpp`](agent_manager_actions.cpp.md) · [`agent_manager_properties.cpp`](agent_manager_properties.cpp.md) · [`agent_member_manager.cpp`](agent_member_manager.cpp.md) · [`agent_member_manager_inline.h`](agent_member_manager_inline.h.md) · [`agent_memory_manager.cpp`](agent_memory_manager.cpp.md) · [`ai_monsters_misc.cpp`](ai/ai_monsters_misc.cpp.md) · [`ai_stalker_cover.cpp`](ai/stalker/ai_stalker_cover.cpp.md) · [`ai_stalker_fire.cpp`](ai/stalker/ai_stalker_fire.cpp.md) · [`ai_stalker_misc.cpp`](ai/stalker/ai_stalker_misc.cpp.md) · _and 20 more_
**Tier floor** — T3: a declaration.

## Purpose

Declares the surface implemented in
[`agent_member_manager.cpp`](agent_member_manager.cpp.md) and in
[`agent_member_manager_inline.h`](agent_member_manager_inline.h.md), plus the names other
squad code uses for the roster: the roster list type, its iterators, and the squad mask
type (shared with the perception layer, which is why it is defined there and only aliased
here).

Exported units:

- **Roster mutation** — `add`, `remove`, `remove_links`, `update`.
- **Lookup** — `member` by stalker, `member` by mask bit, `get_member` by entity
  identifier, `members` in both mutable and read-only forms.
- **Masks** — `mask` by stalker and by identifier, `combat_mask`, `non_combat_members_mask`.
- **Combat registration** — `register_in_combat`, `unregister_in_combat`,
  `registered_in_combat`, `combat_members`.
- **Group interlocks** — `group_behaviour`, `in_detour`, `can_detour`, `cover_detouring`,
  `can_cry_noninfo_phrase`, `can_throw_grenade`, `on_throw_completed`,
  `throw_time_interval` read and write.
