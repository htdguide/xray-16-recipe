# src/xrGame/alife_spawn_registry.h

> Declares the owner of the spawn file, implemented in [`alife_spawn_registry.cpp`](alife_spawn_registry.cpp.md) and [`alife_spawn_registry_spawn.cpp`](alife_spawn_registry_spawn.cpp.md).

**Needs** — [`alife_spawn_registry_header.h`](alife_spawn_registry_header.h.md) · [`xrServer_Objects_ALife_Monsters.h`](../xrServerEntities/xrServer_Objects_ALife_Monsters.h.md) · [`server_entity_wrapper.h`](server_entity_wrapper.h.md) · [`xrAICore/Navigation/game_graph.h`](../xrAICore/Navigation/game_graph.h.md) · [`xrAICore/Navigation/graph_abstract.h`](../xrAICore/Navigation/graph_abstract.h.md) · [`alife_spawn_registry_inline.h`](alife_spawn_registry_inline.h.md)
**Used by** — [`alife_anomalous_zone.cpp`](alife_anomalous_zone.cpp.md) · [`alife_simulator_base.cpp`](alife_simulator_base.cpp.md) · [`alife_simulator_script.cpp`](alife_simulator_script.cpp.md) · [`alife_spawn_registry.cpp`](alife_spawn_registry.cpp.md) · [`alife_spawn_registry_inline.h`](alife_spawn_registry_inline.h.md) · [`alife_spawn_registry_spawn.cpp`](alife_spawn_registry_spawn.cpp.md) · [`alife_storage_manager.cpp`](alife_storage_manager.cpp.md) · [`alife_surge_manager.cpp`](alife_surge_manager.cpp.md) · [`alife_update_manager.cpp`](alife_update_manager.cpp.md) · [`saved_game_wrapper.cpp`](saved_game_wrapper.cpp.md)
**Tier floor** — T3: a declaration only

## Purpose

Declares `CALifeSpawnRegistry`. Substance is split between
[`alife_spawn_registry.cpp`](alife_spawn_registry.cpp.md) (loading, saving, identity
checks) and
[`alife_spawn_registry_spawn.cpp`](alife_spawn_registry_spawn.cpp.md) (the spawn walk);
the artefact placement and the small helpers are in
[`alife_spawn_registry_inline.h`](alife_spawn_registry_inline.h.md).

Two declaration-level facts matter.

**The spawn data is a serializable weighted graph** whose vertices hold wrapped server
entity records and whose vertex identifiers are the spawn identifiers that entities carry
for life. The graph type is the same generic one the navigation code uses; the spawn file
is literally a serialized graph.

**The registry is itself a random source.** It inherits a random stream rather than
holding one, which makes every spawn decision draw from a stream that belongs to the
spawn data and nothing else — separate from the simulation's stream and from the
renderer's. A rebuild should keep the streams separate, because mixing them makes spawn
outcomes depend on how many frames were rendered.

Exported units:

- `CALifeSpawnRegistry` — construct with a configuration section (ignored), destroy.
- `load` in three forms: by spawn file name, from a save stream plus a save file name, and
  the underlying loader with an optional identity to verify against.
- `save` — the identity and the per-record updates.
- `fill_new_spawns` — the spawn walk.
- `header`, `spawns`, `get_spawn_name`, `get_spawn_file` — read access.
- `spawn_id` — resolve an authored spawn-story identifier to a spawn identifier.
- `assign_artefact_position` — place an artefact inside an anomaly.

**Notes** — the class carries two scratch lists used only while computing the root set.
They are members to avoid reallocating; a rebuild makes them local.
