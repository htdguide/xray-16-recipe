# src/xrGame/alife_group_registry.h

> Declares the index of online/offline groups.

**Needs** — [`alife_group_registry.cpp`](alife_group_registry.cpp.md) · [`alife_group_registry_inline.h`](alife_group_registry_inline.h.md) · [`alife_space.h`](../xrServerEntities/alife_space.h.md)
**Used by** — [`base_monster_script.cpp`](ai/monsters/basemonster/base_monster_script.cpp.md) · [`alife_group_registry.cpp`](alife_group_registry.cpp.md) · [`alife_group_registry_inline.h`](alife_group_registry_inline.h.md) · [`alife_monster_abstract.cpp`](alife_monster_abstract.cpp.md) · [`alife_simulator_base.cpp`](alife_simulator_base.cpp.md) · [`alife_simulator_base2.cpp`](alife_simulator_base2.cpp.md) · [`alife_storage_manager.cpp`](alife_storage_manager.cpp.md)
**Tier floor** — T3: a declaration.

## Purpose

Declares the surface implemented in
[`alife_group_registry.cpp`](alife_group_registry.cpp.md): `add` and `remove` with their
self-filtering behaviour, `object` by entity identifier, `objects` for the whole table, and
`on_after_game_load`.
