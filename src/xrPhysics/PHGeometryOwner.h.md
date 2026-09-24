# src/xrPhysics/PHGeometryOwner.h

> Declares the mixin that owns a set of collision shapes as one composite: their grouping, their shared material and callbacks, and the mass properties derived from them.

**Needs** — [`Geometry.h`](Geometry.h.md) · [`PHGeometryOwner.cpp`](PHGeometryOwner.cpp.md)
**Used by** — [`PHElement.cpp`](PHElement.cpp.md) · [`PHElement.h`](PHElement.h.md) · [`PHGeometryOwner.cpp`](PHGeometryOwner.cpp.md) · [`PHMoveStorage.h`](PHMoveStorage.h.md) · [`PHStaticGeomShell.cpp`](PHStaticGeomShell.cpp.md) · [`PHStaticGeomShell.h`](PHStaticGeomShell.h.md) · [`ShellHit.cpp`](ShellHit.cpp.md)
**Tier floor** — T2: shape bookkeeping and volume arithmetic.

## Purpose

Declares the surface implemented in [`PHGeometryOwner.cpp`](PHGeometryOwner.cpp.md). Anything in
the module that owns more than one collision shape and wants them treated as a unit mixes this in:
a rigid-body element (one bone's worth of shapes) and a static geometry proxy both do.

## Exported units

- `CPHGeometryOwner` — the mixin. Full contracts in the implementation twin.
- `GEOM_STORAGE` / `GEOM_I` / `GEOM_CI` — the shape collection and its traversal aliases.
- `t_get_extensions(shapes, axis, origin_offset, OUT low, OUT high)` — the union extent of a
  collection along an axis, as the minimum low and maximum high over its members. Templated so the
  same function serves a collection of shapes and a collection of whole elements; in a rebuild this
  is one function over anything that can report an extent.

## Notes

The mixin holds the shapes *by owning link* and destroys them with itself — the shapes outlive
neither their owner nor the group they were added to. That ownership rule is the only thing about
this type that a rebuild must copy exactly; everything else is bookkeeping.
