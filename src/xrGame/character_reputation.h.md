# src/xrGame/character_reputation.h

> Declares the reputation band value and its shared table, implemented in [`character_reputation.cpp`](character_reputation.cpp.md).

**Needs** — [`character_info_defs.h`](../xrServerEntities/character_info_defs.h.md) · [`ini_id_loader.h`](ini_id_loader.h.md) · [`ini_table_loader.h`](ini_table_loader.h.md)
**Used by** — [`character_reputation.cpp`](character_reputation.cpp.md) · [`relation_registry.cpp`](relation_registry.cpp.md) · [`relation_registry_inline.h`](relation_registry_inline.h.md) · [`xrgame_dll_detach.cpp`](xrgame_dll_detach.cpp.md) · [`character_info.h`](../xrServerEntities/character_info.h.md)
**Tier floor** — T3: a declaration

## Purpose

Declares the surface implemented in
[`character_reputation.cpp`](character_reputation.cpp.md).

Exported units:

- **`REPUTATION_DATA`** — one band's name, index and upper threshold.
- **`CHARACTER_REPUTATION`** — with **`set`**, **`value`**, **`index`**, **`id`** and the
  static **`ValueToIndex`**.
- **`relation`** — the square reputation-to-reputation goodwill table, read-only.
- **`InitIdToIndex`, `DeleteIdToIndexData`** — data-location declaration and teardown.
