# src/xrGame/alife_object_registry.h

> Declares the master table of alife server objects, implemented in [`alife_object_registry.cpp`](alife_object_registry.cpp.md) and [`alife_object_registry_inline.h`](alife_object_registry_inline.h.md).

**Needs** — [`xrServer_Objects_ALife.h`](../xrServerEntities/xrServer_Objects_ALife.h.md) · [`alife_object_registry_inline.h`](alife_object_registry_inline.h.md)
**Used by** — [`monster_state_smart_terrain_task_inline.h`](ai/monsters/states/monster_state_smart_terrain_task_inline.h.md) · [`alife_dynamic_object.cpp`](alife_dynamic_object.cpp.md) · [`alife_group_abstract.cpp`](alife_group_abstract.cpp.md) · [`alife_monster_abstract.cpp`](alife_monster_abstract.cpp.md) · [`alife_object_registry.cpp`](alife_object_registry.cpp.md) · [`alife_object_registry_inline.h`](alife_object_registry_inline.h.md) · [`alife_online_offline_group.cpp`](alife_online_offline_group.cpp.md) · [`alife_simulator.cpp`](alife_simulator.cpp.md) · [`alife_simulator_base.cpp`](alife_simulator_base.cpp.md) · [`alife_simulator_base2.cpp`](alife_simulator_base2.cpp.md) · [`alife_simulator_script.cpp`](alife_simulator_script.cpp.md) · [`alife_storage_manager.cpp`](alife_storage_manager.cpp.md) · [`alife_surge_manager.cpp`](alife_surge_manager.cpp.md) · [`alife_switch_manager.cpp`](alife_switch_manager.cpp.md) · _and 13 more_
**Tier floor** — T3: a declaration only

## Purpose

Declares `CALifeObjectRegistry`: an owning map from entity identifier to alife server
object, plus its save and load. Substance is in
[`alife_object_registry.cpp`](alife_object_registry.cpp.md); the table operations are in
[`alife_object_registry_inline.h`](alife_object_registry_inline.h.md).

The constructor takes a configuration section name and ignores it. Every alife
sub-registry has that signature so the simulator can construct them uniformly; this one
has nothing to configure. A rebuild should drop the parameter here and keep it only where
it is read.

Both the destructor and `save` are overridable, because the multiplayer server
substitutes a registry that saves a different subset.

Exported units:

- `CALifeObjectRegistry` — construct, destroy (unregister-all then delete-all).
- `save` / `load` — the object chunk of a saved game.
- `get_object` — decode one object from a stream, without registering it; also used by
  the network path.
- `add` / `remove` / `object` — table operations.
- `objects` — the whole table, for callers that must iterate.
