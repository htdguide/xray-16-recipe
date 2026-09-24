# src/xrCDB/xrCDB.h

> Declares the public surface of the static collision database — the triangle, the
> result, the query options, the model, the collider and the two soup collectors.

**Needs** — [`xrCore/Threading/Lock.hpp`](../xrCore/Threading/Lock.hpp.md) · [`Common/Noncopyable.hpp`](../Common/Noncopyable.hpp.md) · [`xrCore/_vector3d.h`](../xrCore/_vector3d.h.md) · [`xrCommon/xr_vector.h`](../xrCommon/xr_vector.h.md)
**Used by** — [`DetailManager.h`](../Layers/xrRender/DetailManager.h.md) · [`DetailManager_Decompress.cpp`](../Layers/xrRender/DetailManager_Decompress.cpp.md) · [`FSkinned.cpp`](../Layers/xrRender/FSkinned.cpp.md) · [`HOM.cpp`](../Layers/xrRender/HOM.cpp.md) · [`HOM.h`](../Layers/xrRender/HOM.h.md) · [`LightTrack.cpp`](../Layers/xrRender/LightTrack.cpp.md) · [`LightTrack.h`](../Layers/xrRender/LightTrack.h.md) · [`light_gi.cpp`](../Layers/xrRender/light_gi.cpp.md) · [`r__sector_detect.cpp`](../Layers/xrRender/r__sector_detect.cpp.md) · [`Frustum.h`](Frustum.h.md) · [`ISpatial.h`](ISpatial.h.md) · [`xrCDB.cpp`](xrCDB.cpp.md) · [`xrCDB_Collector.cpp`](xrCDB_Collector.cpp.md) · [`xrCDB_box.cpp`](xrCDB_box.cpp.md) · _and 19 more_
**Tier floor** — T1: it fixes the byte layout of a record that is read straight out of a
level file and written straight into a cache file.

## Purpose

Declares everything the rest of the engine knows about the static collision database. The
substance is in [`xrCDB.cpp`](xrCDB.cpp.md) (the model and its lifecycle),
[`xrCDB_ray.cpp`](xrCDB_ray.cpp.md), [`xrCDB_box.cpp`](xrCDB_box.cpp.md),
[`xrCDB_frustum.cpp`](xrCDB_frustum.cpp.md) (the three traversals) and
[`xrCDB_Collector.cpp`](xrCDB_Collector.cpp.md) (the collectors).

One thing is decided here rather than there: the file forward-declares the vendored tree
type and never exposes it, so no caller outside this directory can see or touch the node
array. That is what makes the delegated part genuinely swappable.

## State

Declares the shapes; the records and their invariants are written down in
[`xrCDB.cpp`](xrCDB.cpp.md), which is where they are maintained.

## Exported units

- **`Triangle`** — the soup's element: three vertex indices plus a packed payload. Layout
  frozen at 16 bytes; fields in [`xrCDB.cpp`](xrCDB.cpp.md).
- **`Model`** — one immutable collision world: geometry, tree, lifecycle, serialization.
- **`Result`** — one hit: the three vertex positions, the triangle's payload word, the
  triangle index, and for a ray also the distance along it and the barycentric coordinates
  of the intersection point.
- **query options** — `cull` (reject back-facing triangles), `only_first` (stop at the
  first hit found, in no particular order), `only_nearest` (keep one hit, the closest),
  `full_test` (for box and frustum: run the exact separating-axis / clipping test at a leaf
  instead of accepting the whole leaf triangle).
- **`Collider`** — the per-thread query handle: holds the result buffer and exposes
  `ray_query`, `box_query`, `frustum_query`.
- **build fix-up hook** — a step invoked on the copied geometry before the tree is built.
- **serialize / deserialize hooks** — let the owner of a cache file add its own validity
  data and its own check of it.
- **material remapping hook** — rewrites triangle material ids when a prebuilt tree is
  loaded but the material library has changed underneath it.
- **`Collector`** — accumulates a soup, optionally welding coincident vertices by linear
  search. For small soups.
- **`CollectorPacked`** — the same, with a fixed spatial hash over a known bounding box so
  that welding is constant-time. For a whole level.

## Notes

`Result` carries the three vertex *positions*, not the triangle index alone, which
duplicates 36 bytes per hit that the caller could have looked up. It is there because most
callers want the positions and the traversal has them in registers already; a rebuild
weighing cache pressure against a second indirection may decide differently, but then the
frustum query — whose callers want the clipped geometry and nothing else — needs another
way to hand them over.
