# src/xrGame/magic_box3.h

> Declares an oriented bounding box: a centre, three orthonormal axes and three half-extents.

**Needs** — [`magic_box3_inline.h`](magic_box3_inline.h.md) · [`magic_box3.cpp`](magic_box3.cpp.md)
**Used by** — [`GameObject.cpp`](GameObject.cpp.md) · [`ai_obstacle.cpp`](ai_obstacle.cpp.md) · [`ai_obstacle.h`](ai_obstacle.h.md) · [`magic_box3.cpp`](magic_box3.cpp.md) · [`magic_box3_inline.h`](magic_box3_inline.h.md) · [`min_obb.cpp`](min_obb.cpp.md) · [`moving_objects_dynamic.cpp`](moving_objects_dynamic.cpp.md) · [`moving_objects_dynamic_collision.cpp`](moving_objects_dynamic_collision.cpp.md)
**Tier floor** — T3: a declaration

## Purpose

Declares `MagicBox3`. See [`magic_box3.cpp`](magic_box3.cpp.md) for the two operations with
substance and [`magic_box3_inline.h`](magic_box3_inline.h.md) for construction and the
accessors.

Exported units:

- `MagicBox3` — centre, three axes, three half-extents.
- construction from nothing, and from a transform plus a half-size.
- `Center` · `Axis` · `Axes` · `Extent` · `Extents` — read and write access to each part.
- `ComputeVertices` — the eight corners.
- `intersects` — overlap against another oriented box.

**Notes** — the name is the original author's, carried over from the geometry library this
was adapted from. It means nothing; a rebuild calls it an oriented bounding box.
