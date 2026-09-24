# src/xrGame/character_rank.h

> Declares the rank band value and its shared tables, implemented in [`character_rank.cpp`](character_rank.cpp.md).

**Needs** — [`character_info_defs.h`](../xrServerEntities/character_info_defs.h.md) · [`ini_id_loader.h`](ini_id_loader.h.md) · [`ini_table_loader.h`](ini_table_loader.h.md)
**Used by** — [`character_rank.cpp`](character_rank.cpp.md) · [`relation_registry.cpp`](relation_registry.cpp.md) · [`relation_registry_inline.h`](relation_registry_inline.h.md) · [`xrgame_dll_detach.cpp`](xrgame_dll_detach.cpp.md) · [`character_info.h`](../xrServerEntities/character_info.h.md)
**Tier floor** — T3: a declaration

## Purpose

Declares the surface implemented in [`character_rank.cpp`](character_rank.cpp.md): a rating
number plus the band it falls in, over the shared name-to-index machinery.

Exported units:

- **`RANK_DATA`** — one band's name, index and upper threshold.
- **`CHARACTER_RANK`** — with **`set`** (by rating), **`value`** (the raw rating),
  **`index`** and **`id`** (the derived band), and **`ValueToIndex`** (the banding itself,
  static and pure).
- **`relation`** — the square rank-to-rank goodwill table, read-only.
- **`rank_kill_points`** — the rating awarded for killing a character of a band.
- **`InitIdToIndex`, `DeleteIdToIndexData`** — data-location declaration and teardown.

**Notes** — the initial state is rating "none" with band index zero, which is *not*
consistent with the banding function for that rating. Nothing reads the band before the first
`set`, but a rebuild should derive the initial band rather than assume zero.
