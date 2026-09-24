# src/xrPhysics/PHMoveStorage.cpp

> Recovers the two world positions bounding one shape's motion this step, from whichever place the collider left them.

**Needs** — [`PHMoveStorage.h`](PHMoveStorage.h.md) · [`Geometry.h`](Geometry.h.md) · [`ExtendedGeom.h`](ExtendedGeom.h.md) · [Seam: Rigid-body dynamics](../../SYSTEM-REQUIREMENTS.md#seam-rigid-body-dynamics)
**Used by** — [`PHMoveStorage.h`](PHMoveStorage.h.md)
**Tier floor** — T1: reads the collider's internal cached transform for an offset-attached shape.

## Purpose

Stateless. One operation, and it exists because the two ways a shape can be attached to a body
store "where is this shape now" in two different places.

A shape attached directly to a body has a world position the collider maintains. A shape attached
through an *offset wrapper* — the usual case, since a bone's shape is rarely centred on its body's
centre of mass — has no such position: the wrapper composes the body's placement with the offset
lazily, caching the result only as a side effect of recomputing its bounding volume.

The previous step's position is recorded by the engine itself, in the shape's user data, and is
available uniformly.

## `positions`

**Contract** — reports the start and end of this step's motion for the shape the iterator names.
For an offset-attached shape it **forces the collider to refresh its cached composed transform**
before reading it, because the cache is only valid alongside a current bounding volume and the
caller may have moved the body since.

```text
FUNCTION positions(shape) -> (from, to)
  IF shape IS attached through an offset wrapper THEN
    shape.wrapper.recompute_bounds()            # side effect: refreshes the composed transform
    from := shape.user_data.previous_position   # recorded by the engine at the last read-back
    to   := shape.wrapper.composed_position
  ELSE
    to   := shape.world_position
    from := shape.user_data.previous_position
```

**Invariants** — the caller must treat a `from` equal to the sentinel "never placed" value as
meaning there is no motion to sweep; the sweep loop in [`PHObject.cpp`](PHObject.cpp.md) checks it
explicitly. A shape on its very first step has no previous position.

**Notes** — the file redeclares the collider's internal offset-wrapper record in order to read the
cached transform. That is a layering violation the original accepted to avoid a per-shape
composition; a rebuild that owns its own collider should simply expose the composed transform, and
this file shrinks to two lines.

The reliance on a refresh that happens *as a side effect* of a bounds recomputation is the fragile
part, and it is worth stating as a requirement rather than reproducing the mechanism: **a swept
query needs the shape's current world placement, and the rebuild must have a way to demand one.**
