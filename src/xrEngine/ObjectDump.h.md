# src/xrEngine/ObjectDump.h

> Declares the six object-to-text dumps used when a diagnostic needs to say which object it means.

**Needs** — [`ObjectDump.cpp`](ObjectDump.cpp.md) · [`xr_object.h`](xr_object.h.md)
**Used by** — [`ObjectDump.cpp`](ObjectDump.cpp.md)
**Tier floor** — T3: string formatting

## Purpose

Declares the surface implemented in [`ObjectDump.cpp`](ObjectDump.cpp.md). Development
builds only; the whole surface compiles away in a shipping build, and so must every call
site.

Exported units — each renders one facet of a game object as text, and each tolerates a
missing object:

- `dbg_object_base_dump_string` — identity: instance name, configuration section, visual asset.
- `dbg_object_props_dump_string` — the object's state bits and its three update frame stamps.
- `dbg_object_poses_dump_string` — current transform plus the object's recent position history.
- `dbg_object_visual_geom_dump_string` — bounding box, centre and radius of the visual.
- `dbg_object_full_dump_string` — all four, in that order.
- `dbg_object_full_capped_dump_string` — the same with a leading banner line.
