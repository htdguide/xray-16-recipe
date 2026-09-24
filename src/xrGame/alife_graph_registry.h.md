# src/xrGame/alife_graph_registry.h

> Declares the offline world's index by game graph vertex, its terrain transpose and its level-scoped subset.

**Needs** — [`alife_graph_registry.cpp`](alife_graph_registry.cpp.md) · [`alife_graph_registry_inline.h`](alife_graph_registry_inline.h.md) · [`alife_level_registry.h`](alife_level_registry.h.md) · [`xrServerEntities/xrServer_Objects_ALife_All.h`](../xrServerEntities/xrServer_Objects_ALife_All.h.md)
**Used by** — [`LevelGraphDebugRender.cpp`](LevelGraphDebugRender.cpp.md) · [`alife_anomalous_zone.cpp`](alife_anomalous_zone.cpp.md) · [`alife_combat_manager.cpp`](alife_combat_manager.cpp.md) · [`alife_dynamic_object.cpp`](alife_dynamic_object.cpp.md) · [`alife_graph_registry.cpp`](alife_graph_registry.cpp.md) · [`alife_graph_registry_inline.h`](alife_graph_registry_inline.h.md) · [`alife_group_abstract.cpp`](alife_group_abstract.cpp.md) · [`alife_monster_abstract.cpp`](alife_monster_abstract.cpp.md) · [`alife_monster_detail_path_manager.cpp`](alife_monster_detail_path_manager.cpp.md) · [`alife_online_offline_group.cpp`](alife_online_offline_group.cpp.md) · [`alife_simulator_base.cpp`](alife_simulator_base.cpp.md) · [`alife_simulator_base2.cpp`](alife_simulator_base2.cpp.md) · [`alife_simulator_script.cpp`](alife_simulator_script.cpp.md) · [`alife_storage_manager.cpp`](alife_storage_manager.cpp.md) · _and 6 more_
**Tier floor** — T3: a declaration.

## Purpose

Declares the surface implemented in
[`alife_graph_registry.cpp`](alife_graph_registry.cpp.md) and
[`alife_graph_registry_inline.h`](alife_graph_registry_inline.h.md).

Exported units:

- **`on_load`** — size the index to a freshly loaded game graph and build the terrain
  transpose;
- **`update`** — register one object and, on first sight of the player, load the level;
- **`add`**, **`remove`**, **`change`** — index maintenance;
- **`attach`**, **`detach`** — an item changing hands, index and ownership together;
- **`assign`** — seed a creature's graph-travel state;
- **`iterate_objects`** — visit every object at one graph vertex, safely against mutation;
- **`level`**, **`actor`**, **`objects`**, **`set_process_time`** — accessors.

**Notes** — The per-vertex tables are declared as a *mutation-safe* keyed table: the
iteration form takes a callback rather than handing out iterators, because the callbacks
the simulation runs at a vertex routinely add and remove objects at that same vertex. A
rebuild must provide the same guarantee — iterate over a snapshot, or defer structural
changes — or the simulation will invalidate its own iterator on the first interesting
event.

The terrain index is two-dimensional: a fixed number of classification axes by a fixed
number of values per axis. Both counts come from the game graph's own format and are
therefore frozen.
