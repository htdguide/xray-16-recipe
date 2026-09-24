# src/xrGame/space_restriction.h

> Declares one entity's composed movement restriction: a permitted volume, a forbidden volume, and the merged border between them.

**Needs** — [`space_restriction_abstract.h`](space_restriction_abstract.h.md) · [`space_restriction_holder.h`](space_restriction_holder.h.md) · [`space_restriction_bridge.h`](space_restriction_bridge.h.md) · [`space_restriction_inline.h`](space_restriction_inline.h.md) · [`xrServerEntities/restriction_space.h`](../xrServerEntities/restriction_space.h.md)
**Used by** — [`restricted_object.cpp`](restricted_object.cpp.md) · [`space_restriction.cpp`](space_restriction.cpp.md) · [`space_restriction_inline.h`](space_restriction_inline.h.md) · [`space_restriction_manager.cpp`](space_restriction_manager.cpp.md) · [`space_restriction_manager.h`](space_restriction_manager.h.md) · [`space_restriction_manager_inline.h`](space_restriction_manager_inline.h.md)
**Tier floor** — T2: a declaration over vertex lists

## Purpose

Declares the surface implemented in [`space_restriction.cpp`](space_restriction.cpp.md) and
[`space_restriction_inline.h`](space_restriction_inline.h.md). The type is both a
reference-counted, garbage-collectable record (so that entities with identical restriction
lists share one) and an implementor of the restriction interface (so that it can supply a
border and a name like any other restriction).

## Exported units

- `initialize` — resolve the two name lists and merge their borders.
- `accessible(sphere)`, `accessible(vertex, radius)` — the two in/out tests.
- `accessible_nearest(position)` — nearest legal vertex and point.
- `add_border`, `remove_border` — stamp and clear the merged border on the level graph.
- `out_restrictions`, `in_restrictions`, `applied`, `initialized` — accessors.
- `inside(sphere)`, `inside(vertex, partially)` — containment in the *permitted* space.
- `name` — the out-restriction list.
- `affect` — vestigial relevance tests; see the implementation twin.

## Notes

The lazily-enabled-forbidden-volume policy declares a small record here pairing a
restriction with an enabled flag, and a list of them. Both are compiled out with that
policy; see [`space_restriction.cpp`](space_restriction.cpp.md).
