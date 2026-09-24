# src/xrCDB/xr_area_query.cpp

> Answers an *oriented*-box query against the static world by expressing the box as
> six half-spaces and running the frustum query.

**Needs** — [`xr_area.h`](xr_area.h.md) · [`Frustum.h`](Frustum.h.md) · [`xrCDB.h`](xrCDB.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: vector and plane construction.

## Purpose

The tree's box query takes an *axis-aligned* box. Callers who have a box with an orientation
— a vehicle's footprint, a door's swept volume, an editor's selection — would have to pass
its axis-aligned bound and then filter, which over-collects badly for a box rotated
forty-five degrees. This file avoids that by noticing that an oriented box is a convex
volume of six planes, which the frustum query already handles exactly.

That is the whole file: a coordinate change and a delegation.

## State

Stateless.

## `ObjectSpace.box_query`

**Contract** — collects the static triangles overlapping an oriented box, given its centre,
its forward and up axes, and its full sizes along each of its three axes. Appends each
surviving triangle's three vertices to a caller-supplied list, if one is given, and returns
whether anything was found. Runs on the calling thread's query handle, so it is safe from
any thread.

```text
FUNCTION box_query(centre, forward, up, sizes, out) -> bool
  k <- normalize(forward)
  j <- normalize(up)
  i <- normalize(cross(up, forward))        # completes the frame

  planes <- the six planes through centre +/- (axis * size/2), each facing outward
  frustum <- from_planes(planes)

  query_handle.frustum_query(frustum, exact)       # exact: clip each triangle
  IF out is given: FOR EACH hit: append its three vertices to out
  RETURN query_handle.result_count > 0
```

**The third axis is derived, not supplied**, so the caller gives two axes and the handedness
falls out of the cross product. The caller's two axes need not be exactly perpendicular —
they are normalized independently and only the cross product's direction is used — but a
badly skewed pair produces a box that is not the one the caller meant, and nothing checks.

**The query asks for the exact test**, not the conservative one. That is the point of the
exercise: a conservative frustum query accepts a whole leaf whose box merely touches the
volume, which would reintroduce the over-collection the oriented box was meant to avoid.

**The results are flattened to loose vertices**, three per triangle, discarding the triangle
index and the payload. The caller here is the physics bridge, which wants a vertex soup to
hand to its own mesh collider and has no use for material or sector.

## Notes

Nothing in this file is bounded. An oriented box the size of a building returns every
triangle in it.
