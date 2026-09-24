# src/xrGame/alife_smart_terrain_registry.h

> Declares the index of smart terrains, implemented in [`alife_smart_terrain_registry.cpp`](alife_smart_terrain_registry.cpp.md).

**Needs** — [`alife_space.h`](../xrServerEntities/alife_space.h.md) · [`alife_smart_terrain_registry_inline.h`](alife_smart_terrain_registry_inline.h.md)
**Used by** — [`alife_simulator_base.cpp`](alife_simulator_base.cpp.md) · [`alife_simulator_base2.cpp`](alife_simulator_base2.cpp.md) · [`alife_smart_terrain_registry.cpp`](alife_smart_terrain_registry.cpp.md) · [`alife_smart_terrain_registry_inline.h`](alife_smart_terrain_registry_inline.h.md)
**Tier floor** — T3: a declaration only

## Purpose

Declares `CALifeSmartTerrainRegistry`: a map from entity identifier to smart zone, holding
every smart terrain in the world across every level. Substance is in
[`alife_smart_terrain_registry.cpp`](alife_smart_terrain_registry.cpp.md).

Unlike the object registry, this one **does not own** its entries — a smart terrain is a
server object owned by the object registry and merely indexed here. Destroying this
registry destroys nothing, which is why it has no explicit teardown.

Exported units:

- `CALifeSmartTerrainRegistry` — a map, default-constructed.
- `add` / `remove` — offered every entity; keeps the smart zones.
- `object` — resolve an identifier to a smart terrain; absence is a fault.
- `objects` — the whole index, for callers that must sweep it.
