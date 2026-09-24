# src/xrGame/hit_memory_manager.h

> Declares the third sense: memory of who has hurt this creature and from where.

**Needs** — [`memory_space.h`](memory_space.h.md) · [`hit_memory_manager_inline.h`](hit_memory_manager_inline.h.md)
**Used by** — [`agent_enemy_manager.cpp`](agent_enemy_manager.cpp.md) · [`enemy_manager.cpp`](enemy_manager.cpp.md) · [`group_hierarchy_holder.cpp`](group_hierarchy_holder.cpp.md) · [`hit_memory_manager.cpp`](hit_memory_manager.cpp.md) · [`hit_memory_manager_inline.h`](hit_memory_manager_inline.h.md) · [`memory_manager.cpp`](memory_manager.cpp.md) · [`memory_manager.h`](memory_manager.h.md) · [`script_game_object2.cpp`](script_game_object2.cpp.md) · [`sound_memory_manager.cpp`](sound_memory_manager.cpp.md) · [`stalker_combat_actions.cpp`](stalker_combat_actions.cpp.md) · [`stalker_search_actions.cpp`](stalker_search_actions.cpp.md) · [`stalker_sound_data_visitor.cpp`](stalker_sound_data_visitor.cpp.md)
**Tier floor** — T2: a declaration

## Purpose

Declares the surface implemented in
[`hit_memory_manager.cpp`](hit_memory_manager.cpp.md) and
[`hit_memory_manager_inline.h`](hit_memory_manager_inline.h.md). Sight, hearing and hit
memory are the three senses, and each has a manager of this shape; the differences are in
what an entry records and what causes one.

Exported units:

- `CHitMemoryManager` — built against the creature it serves, plus the same creature as a
  stalker when it is one (the stalker-only path is squad mask bookkeeping).
- **Lifecycle** — load, reinitialize, reload tuning, per-frame update.
- **Recording** — three entry points that add a hit: from an attacker alone, from a full hit
  description, and from an already-built record.
- **Queries** — the whole list, the record for a given attacker, the last attacker's
  identifier and the time of that hit.
- **Editing** — enable or disable one record, remove one, and drop every reference to a
  departing object.
- **Persistence** — save, load, and the deferred-resolution callback the load path installs.
- `set_squad_objects` — redirects this creature's hit memory into a shared list. The single
  most consequential call on the class; see
  [`group_hierarchy_holder.cpp`](group_hierarchy_holder.cpp.md).

**Notes** — a debug-only "currently selected hit" exists for the AI inspector and is not part
of the behaviour.
