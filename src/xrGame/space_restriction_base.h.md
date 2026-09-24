# src/xrGame/space_restriction_base.h

> Declares the layer that turns "is this sphere inside the volume" into "is this navigation vertex inside the volume", and fixes the ordering the border list is kept in.

**Needs** — [`space_restriction_abstract.h`](space_restriction_abstract.h.md) · [`space_restriction_base.cpp`](space_restriction_base.cpp.md) · [`space_restriction_base_inline.h`](space_restriction_base_inline.h.md)
**Used by** — [`restricted_object.cpp`](restricted_object.cpp.md) · [`space_restriction.cpp`](space_restriction.cpp.md) · [`space_restriction_base.cpp`](space_restriction_base.cpp.md) · [`space_restriction_base_inline.h`](space_restriction_base_inline.h.md) · [`space_restriction_bridge.cpp`](space_restriction_bridge.cpp.md) · [`space_restriction_bridge.h`](space_restriction_bridge.h.md) · [`space_restriction_bridge_inline.h`](space_restriction_bridge_inline.h.md) · [`space_restriction_composition.cpp`](space_restriction_composition.cpp.md) · [`space_restriction_composition.h`](space_restriction_composition.h.md) · [`space_restriction_shape.cpp`](space_restriction_shape.cpp.md) · [`space_restriction_shape.h`](space_restriction_shape.h.md)
**Tier floor** — T2: five sphere tests per vertex query, on the pathfinding path

## Purpose

Declares the surface implemented in
[`space_restriction_base.cpp`](space_restriction_base.cpp.md). It adds one required
operation to the abstract interface — *is this sphere inside me* — and implements
everything else in terms of it, which is why every concrete restriction only has to answer
a geometric question and never a navigation one.

Exported units:

- `inside(vertex, partially)` and `inside(vertex, partially, radius)` — the vertex test.
- `inside(sphere)` — required of an implementor; the only geometry an implementor owns.
- `shape()` — required: is this a single authored volume, or a composition?
- `default_restrictor()` — required: does this volume apply to everyone by default?
- `sphere()` — required: a bounding sphere, used to reject compositions cheaply.
- `process_borders()` — sort and deduplicate the border into its canonical order.

## State

```text
# adds no fields of its own beyond the abstract base, except a checked-build
# record of every vertex found fully inside, used to verify border connectivity
```
