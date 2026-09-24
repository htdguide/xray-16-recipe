# src/xrGame/alife_simulator.h

> Declares the concrete alife simulator, implemented in [`alife_simulator.cpp`](alife_simulator.cpp.md).

**Needs** — [`alife_update_manager.h`](alife_update_manager.h.md) · [`alife_interaction_manager.h`](alife_interaction_manager.h.md)
**Used by** — [`Entity.cpp`](Entity.cpp.md) · [`GameTask.cpp`](GameTask.cpp.md) · [`LevelGraphDebugRender.cpp`](LevelGraphDebugRender.cpp.md) · [`base_monster_script.cpp`](ai/monsters/basemonster/base_monster_script.cpp.md) · [`base_monster_startup.cpp`](ai/monsters/basemonster/base_monster_startup.cpp.md) · [`monster_state_manager_inline.h`](ai/monsters/monster_state_manager_inline.h.md) · [`monster_state_smart_terrain_task_graph_walk_inline.h`](ai/monsters/states/monster_state_smart_terrain_task_graph_walk_inline.h.md) · [`monster_state_smart_terrain_task_inline.h`](ai/monsters/states/monster_state_smart_terrain_task_inline.h.md) · [`ai_space.cpp`](ai_space.cpp.md) · [`ai_stalker_alife.cpp`](ai_stalker_alife.cpp.md) · [`alife_anomalous_zone.cpp`](alife_anomalous_zone.cpp.md) · [`alife_creature_abstract.cpp`](alife_creature_abstract.cpp.md) · [`alife_dynamic_object.cpp`](alife_dynamic_object.cpp.md) · [`alife_group_abstract.cpp`](alife_group_abstract.cpp.md) · _and 33 more_
**Tier floor** — T3: a declaration only

## Purpose

Declares `CALifeSimulator`, the assembly at the top of the alife layer. Substance is in
[`alife_simulator.cpp`](alife_simulator.cpp.md).

The shape worth carrying over is the **layered assembly**: the simulator is built by
stacking responsibilities, each layer adding one concern and depending only on the layers
below it —

1. a header holding the world's identity and its version;
2. a base holding the registries, the graph, and the time, storage and switch managers;
3. an update manager adding the per-frame loop, the spawn and save paths;
4. an interaction manager adding the offline encounter rules;
5. this class adding the session lifecycle and the configuration cache.

In the original the stack is expressed as an inheritance chain, which makes every layer's
members directly visible to every layer above and gives the whole thing one identity. A
rebuild can compose instead of inherit, and should, but must then decide explicitly which
layer owns which registry — that ownership is currently implicit in the order of the
chain, and several layers reach across it freely.

Exported units:

- `CALifeSimulator` — construct (starts a game), `destroy` (ends one), destruct.
- `get_config` — session-lifetime access to a configuration file by name.
- `setup_simulator` — stamp an entity with its simulator.
- `reload` — re-read the tuning section.
- A script registration hook; surface in
  [`alife_simulator_script.cpp`](alife_simulator_script.cpp.md).
