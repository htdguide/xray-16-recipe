# src/xrEngine/xr_object_list.h

> Declares the per-level object registry; the substance is in [`xr_object_list.cpp`](xr_object_list.cpp.md).

**Needs** — [`xr_object_list.cpp`](xr_object_list.cpp.md) · [`xr_object.h`](xr_object.h.md) · [`GameFont.h`](GameFont.h.md)
**Used by** — [`IGame_Level.cpp`](IGame_Level.cpp.md) · [`IGame_Level.h`](IGame_Level.h.md) · [`pure_relcase.cpp`](pure_relcase.cpp.md) · [`xr_object_list.cpp`](xr_object_list.cpp.md)
**Tier floor** — T2.

## Purpose

Declares the surface described in [`xr_object_list.cpp`](xr_object_list.cpp.md), plus one
record it does not: the per-frame update statistics block (update timer, crows updated,
active count, total count) that the debug overlay reads.

Exported units:

- **`CObjectList`** — the registry itself. Membership (`Create`, `Destroy`, `Load`,
  `Unload`, the find-by-name and find-by-class queries), the frame pass (`Update`), the
  crow marks (`o_crow`, `o_activate`, `o_sleep`), enumeration (`o_count`,
  `o_get_by_iterator`), the network id map (`net_Register`, `net_Unregister`, `net_Find`,
  `net_Export`, `net_Import`), deferred destruction (`register_object_to_destroy`), the
  relcase hook table (`relcase_register`, `relcase_unregister`), and the statistics
  readout.
- **`CObjectList::ObjectUpdateStatistics`** — the frame counters, reset at frame start.
- **`CObjectList::SRelcasePair`** — one relcase hook: the callback and the back-pointer to
  the subscriber's own slot field, which the registry keeps accurate.
- A debug-build flag that turns on a log line per destroyed object.
