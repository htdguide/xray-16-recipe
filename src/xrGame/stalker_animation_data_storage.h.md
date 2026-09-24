# src/xrGame/stalker_animation_data_storage.h

> Declares the shared cache of loaded stalker animation tables, keyed by motion-bank list.

**Needs** — [`stalker_animation_data_storage_inline.h`](stalker_animation_data_storage_inline.h.md)
**Used by** — [`stalker_animation_data_storage.cpp`](stalker_animation_data_storage.cpp.md) · [`stalker_animation_data_storage_inline.h`](stalker_animation_data_storage_inline.h.md) · [`stalker_animation_manager.cpp`](stalker_animation_manager.cpp.md)
**Tier floor** — T3: a declaration over a small cache

## Purpose

Declares the surface implemented in [`stalker_animation_data_storage.cpp`](stalker_animation_data_storage.cpp.md)
and [`stalker_animation_data_storage_inline.h`](stalker_animation_data_storage_inline.h.md).

## Exported units

- `object(skeleton)` — the animation table for a skeleton's bank list, loaded on first ask.
- `clear` — destroy every loaded table.
- The process-wide accessor that creates the storage on first use.

## Notes

Each entry pairs a table with the skeleton that first asked for it; that skeleton is kept
only as a witness for the bank-list comparison, not as an owner. The distinction matters
for lifetime: the witness must outlive the cache.
