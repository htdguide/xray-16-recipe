# src/xrAICore/Navigation/PatrolPath/patrol_path_storage.h

> Declares the registry of every patrol path on a level, keyed by name, implemented in [`patrol_path_storage.cpp`](patrol_path_storage.cpp.md).

**Needs** — [`patrol_path_storage.cpp`](patrol_path_storage.cpp.md) · [`patrol_path_storage_inline.h`](patrol_path_storage_inline.h.md) · [`patrol_path.h`](patrol_path.h.md) · [`../../../Common/object_interfaces.h`](../../../Common/object_interfaces.h.md) · [`../../../xrCore/Containers/AssociativeVector.hpp`](../../../xrCore/Containers/AssociativeVector.hpp.md)
**Used by** — [`AISpaceBase.cpp`](../../AISpaceBase.cpp.md) · [`patrol_path_params.cpp`](patrol_path_params.cpp.md) · [`patrol_path_storage.cpp`](patrol_path_storage.cpp.md) · [`patrol_path_storage_inline.h`](patrol_path_storage_inline.h.md) · [`HelicopterMovementManager.cpp`](../../../xrGame/HelicopterMovementManager.cpp.md) · [`monster_home.cpp`](../../../xrGame/ai/monsters/monster_home.cpp.md) · [`alife_monster_patrol_path_manager.cpp`](../../../xrGame/alife_monster_patrol_path_manager.cpp.md) · [`alife_smart_terrain_task.cpp`](../../../xrGame/alife_smart_terrain_task.cpp.md) · [`patrol_path_manager.h`](../../../xrGame/patrol_path_manager.h.md)
**Tier floor** — T2: an ordered name-keyed registry that serializes.

## Purpose

Declares the type implemented in [`patrol_path_storage.cpp`](patrol_path_storage.cpp.md) and
[`patrol_path_storage_inline.h`](patrol_path_storage_inline.h.md). Patrol paths are addressed by
name from script and from spawn data, so they need a name-keyed registry; this is it, and it
owns the paths.

## Exported units

- `load_raw(level_graph, cross_table, game_graph, stream)` — read the level editor's authored
  form, converting each path onto the navigation mesh as it goes
- `load(stream)` / `save(stream)` — read and write the pre-converted runtime form
- `path(name, tolerate_absence)` — look one up
- `patrol_paths()` — the whole registry
- `add_alias_if_exist(name, alias)` — make an existing path answer to a second name

**Notes** — the registry is a *sorted sequence* keyed by name rather than a hash table. Lookups
are binary searches over interned names; the count is in the hundreds per level and the
sequence's locality beats a hash table at that size. A rebuild may use either.
