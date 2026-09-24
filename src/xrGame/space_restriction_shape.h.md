# src/xrGame/space_restriction_shape.h

> Declares the restriction that wraps one restrictor entity's collision volume.

**Needs** — [`space_restriction_base.h`](space_restriction_base.h.md) · [`space_restriction_shape_inline.h`](space_restriction_shape_inline.h.md) · [`xrEngine/xr_collide_form.h`](../xrEngine/xr_collide_form.h.md)
**Used by** — [`space_restriction_holder.cpp`](space_restriction_holder.cpp.md) · [`space_restriction_shape.cpp`](space_restriction_shape.cpp.md) · [`space_restriction_shape_inline.h`](space_restriction_shape_inline.h.md)
**Tier floor** — T2: a declaration over a reference to a live entity

## Purpose

Declares the surface implemented in [`space_restriction_shape.cpp`](space_restriction_shape.cpp.md)
and [`space_restriction_shape_inline.h`](space_restriction_shape_inline.h.md).

## Exported units

- `initialize` — a no-op assertion; a shape is initialized from construction.
- `inside(sphere)` — delegates to the restrictor entity's exact volume test.
- `name` — the restrictor entity's name.
- `shape` — always true, which is what exempts shapes from garbage collection.
- `default_restrictor` — whether this restrictor is in one of the level's default lists.
- `sphere` — the restrictor's world-space bounding sphere.
- `build_border`, `fill_shape` — border construction, run from the constructor.
- `test_correctness` — checked-build border connectivity check.

## Notes

The constant `shape` answer is the predicate the registry's garbage collector uses. Shapes
are owned by live restrictor entities and must never be reclaimed; everything else in the
registry may be.
