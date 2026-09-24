# src/xrGame/character_community.h

> Declares the community value and its shared tables, implemented in [`character_community.cpp`](character_community.cpp.md).

**Needs** — [`character_info_defs.h`](../xrServerEntities/character_info_defs.h.md) · [`ini_id_loader.h`](ini_id_loader.h.md) · [`ini_table_loader.h`](ini_table_loader.h.md)
**Used by** — [`controller.cpp`](ai/monsters/controller/controller.cpp.md) · [`character_community.cpp`](character_community.cpp.md) · [`relation_registry.cpp`](relation_registry.cpp.md) · [`relation_registry_inline.h`](relation_registry_inline.h.md) · [`xrgame_dll_detach.cpp`](xrgame_dll_detach.cpp.md) · [`character_info.h`](../xrServerEntities/character_info.h.md)
**Tier floor** — T3: a declaration

## Purpose

Declares the surface implemented in
[`character_community.cpp`](character_community.cpp.md): a small value type over the
name-to-index machinery, plus the two global tables it indexes.

Exported units:

- **`COMMUNITY_DATA`** — one community's authored name, its assigned index, and the
  multiplayer team it maps to.
- **`CHARACTER_COMMUNITY`** — the per-character value, with **`set`** (by name or by index),
  **`id`**, **`index`** and **`team`**.
- **`relation`, `set_relation`** — read and write the square inter-community goodwill table.
- **`sympathy`** — the per-community propagation coefficient.
- **`InitIdToIndex`, `DeleteIdToIndexData`** — tell the generic index machinery where the
  data lives, and tear it down.
