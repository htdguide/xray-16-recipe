# src/xrGame/sound_collection_storage.h

> Declares the process-wide table that makes identical sound sets load once and be shared by every creature that uses them.

**Needs** — [`sound_player.h`](sound_player.h.md) · [`sound_collection_storage.cpp`](sound_collection_storage.cpp.md) · [`sound_collection_storage_inline.h`](sound_collection_storage_inline.h.md)
**Used by** — [`sound_collection_storage.cpp`](sound_collection_storage.cpp.md) · [`sound_collection_storage_inline.h`](sound_collection_storage_inline.h.md) · [`sound_player.cpp`](sound_player.cpp.md) · [`xrgame_dll_detach.cpp`](xrgame_dll_detach.cpp.md)
**Tier floor** — T2: a shared cache holding device-backed audio handles

## Purpose

Declares the surface implemented in
[`sound_collection_storage.cpp`](sound_collection_storage.cpp.md). The key idea is in the
type of the key: a sound collection is identified by the *value* of its parameters — name
stems, voice prefix, variant count, AI sound type — not by any name or handle, so two
creatures configured identically share one loaded set without either knowing about the
other.

Exported units:

- `object(params)` — get or build the collection for a parameter value.
- the destructor — release every collection.
- `sound_collection_storage()` — reach the single instance, creating it on first use; see
  [`sound_collection_storage_inline.h`](sound_collection_storage_inline.h.md).

## State

```text
RECORD sound_collection_storage
  objects : list<(sound_collection_params, sound_collection)>
  # invariant: no two entries have equal params
  # invariant: entries live until the storage dies — there is no eviction
```
