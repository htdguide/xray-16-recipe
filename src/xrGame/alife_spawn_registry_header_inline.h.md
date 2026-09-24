# src/xrGame/alife_spawn_registry_header_inline.h

> The five read accessors of the spawn file's header.

**Needs** — [`alife_spawn_registry_header.h`](alife_spawn_registry_header.h.md)
**Used by** — [`alife_spawn_registry_header.h`](alife_spawn_registry_header.h.md)
**Tier floor** — T3: field reads

## Purpose

Bodies for the header's accessors, split out as a C++ habit. Substance is in
[`alife_spawn_registry_header.cpp`](alife_spawn_registry_header.cpp.md).

## The accessors

**Contract** — `version`, `guid`, `graph_guid`, `count` and `level_count` each return the
corresponding field. Unguarded: unlike the simulator's accessors there is no
initialization check, because a header that has not been loaded is never reachable — the
spawn registry loads it as its first action.
