# src/xrGame/alife_spawn_registry_header.h

> Declares the spawn file's header record, implemented in [`alife_spawn_registry_header.cpp`](alife_spawn_registry_header.cpp.md).

**Needs** — [`Common/LevelStructure.hpp`](../Common/LevelStructure.hpp.md) · [`alife_spawn_registry_header_inline.h`](alife_spawn_registry_header_inline.h.md)
**Used by** — [`alife_spawn_registry.cpp`](alife_spawn_registry.cpp.md) · [`alife_spawn_registry.h`](alife_spawn_registry.h.md) · [`alife_spawn_registry_header.cpp`](alife_spawn_registry_header.cpp.md) · [`alife_spawn_registry_header_inline.h`](alife_spawn_registry_header_inline.h.md)
**Tier floor** — T3: a declaration only

## Purpose

Declares `CALifeSpawnHeader`. Substance is in
[`alife_spawn_registry_header.cpp`](alife_spawn_registry_header.cpp.md).

Read-only by design: the record has a loader and no writer, because the spawn file is data
the engine consumes and never produces. The loader is overridable so that offline tools
building spawn files can extend it.

Exported units:

- `CALifeSpawnHeader` — destroy only; there is no interesting construction.
- `load` — read the six fields with the version checked first.
- `version`, `guid`, `graph_guid`, `count`, `level_count` — read access.
