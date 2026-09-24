# src/xrGame/ObjectDump.h

> Declares the object-state text dumpers implemented in [`ObjectDump.cpp`](ObjectDump.cpp.md).

**Needs** — [`ObjectDump.cpp`](ObjectDump.cpp.md)
**Used by** — [`ObjectDump.cpp`](ObjectDump.cpp.md)
**Tier floor** — T3: a declaration only

## Purpose

Declares the debug formatting entry points. The whole file — declarations and definitions
alike — exists only in non-shipping builds, so every caller is already inside such a guard and
a rebuild may drop it. Substance is in [`ObjectDump.cpp`](ObjectDump.cpp.md).

Exported units:

- `dbg_object_full_dump_string` / `dbg_object_full_capped_dump_string` — everything about one
  object.
- `dbg_object_base_dump_string` — instance name, configuration section, visual model.
- `dbg_object_props_dump_string` — registration flags and update-frame counters.
- `dbg_object_poses_dump_string` — current transform and recent position history.
- `dbg_object_visual_geom_dump_string` — bounding box, centre and radius.
- `get_string` for a boolean, a vector, a transform and a box.
