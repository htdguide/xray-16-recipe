# src/xrGame/alife_simulator_base.h

> Declares the layer that owns the alife registries and entity creation, implemented in [`alife_simulator_base.cpp`](alife_simulator_base.cpp.md) and [`alife_simulator_base2.cpp`](alife_simulator_base2.cpp.md).

**Needs** — [`alife_space.h`](../xrServerEntities/alife_space.h.md) · [`Random.hpp`](Random.hpp.md) · [`xrAICore/Navigation/game_graph_space.h`](../xrAICore/Navigation/game_graph_space.h.md) · [`alife_simulator_base_inline.h`](alife_simulator_base_inline.h.md)
**Used by** — [`alife_combat_manager.cpp`](alife_combat_manager.cpp.md) · [`alife_combat_manager.h`](alife_combat_manager.h.md) · [`alife_communication_manager.cpp`](alife_communication_manager.cpp.md) · [`alife_communication_manager.h`](alife_communication_manager.h.md) · [`alife_simulator.cpp`](alife_simulator.cpp.md) · [`alife_simulator_base.cpp`](alife_simulator_base.cpp.md) · [`alife_simulator_base2.cpp`](alife_simulator_base2.cpp.md) · [`alife_simulator_base_inline.h`](alife_simulator_base_inline.h.md) · [`alife_storage_manager.h`](alife_storage_manager.h.md) · [`alife_surge_manager.h`](alife_surge_manager.h.md) · [`alife_switch_manager.h`](alife_switch_manager.h.md)
**Tier floor** — T3: a declaration only

## Purpose

Declares `CALifeSimulatorBase`. Substance is in
[`alife_simulator_base.cpp`](alife_simulator_base.cpp.md) (registries, creation) and
[`alife_simulator_base2.cpp`](alife_simulator_base2.cpp.md) (registration, death); the
accessors are in
[`alife_simulator_base_inline.h`](alife_simulator_base_inline.h.md).

Two declaration-level decisions matter to a rebuild.

**Access is split into protected and public halves, and the split is the intended API.**
The graph registry, the schedule registry, the time manager, the persistent registry
container and the inventory-upgrade manager are public — the game and the scripts reach
them directly. The header, the spawn registry, the object registry, the story registry,
the smart-terrain registry and the group registry are protected — reachable only from
layers above in the same stack. A rebuild that publishes all eleven loses a real boundary:
those six are mutated only through the registration routines, and direct access to them
is how an entity ends up in one registry and not another.

**The entity-creation hook is abstract.** The base declares that *something* must stamp
each created entity with its simulator, and leaves it to the concrete simulator. That is
the one place the base admits it is not the whole simulator.

Exported units:

- `CALifeSimulatorBase` — construct against a server and a configuration section;
  `destroy`; `initialized`.
- The eleven sub-object accessors, readable and (for most) writable — see the inline
  twin.
- `register_object` / `unregister_object` / `release` — the lifecycle.
- `create` in three forms (from a spawn record, from a group template, adopting a
  client-created object) and `spawn_item` (from a configuration section).
- `on_death` — the simulation's death response.
- `assign_death_position` — where an offline corpse lands.
- `append_item_vector`, `level_name`, `random`, `server`, `server_command_line`.
- `can_register_objects` — the gate that defers entity registration hooks during a bulk
  load.
- A type-keyed shortcut to one persistent registry, forwarding to the registry container.

**Notes** — the header carries a scratch list of item references and two scratch vectors
for offline combat grouping, both marked temporary in the original and both effectively
per-call scratch space hoisted to member scope to avoid reallocation. A rebuild should
make them local; nothing reads them across calls.

The inline accessor bodies are included from the header in optimized builds and compiled
once in debug builds, which is a build-time trade with no behavioural content.
