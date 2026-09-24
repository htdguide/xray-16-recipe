# src/xrGame/space_restriction_bridge.h

> Declares the replaceable, reference-counted cell that every restriction handle points at.

**Needs** — [`space_restriction_base.h`](space_restriction_base.h.md) · [`space_restriction_bridge_inline.h`](space_restriction_bridge_inline.h.md) · [`xrServerEntities/restriction_space.h`](../xrServerEntities/restriction_space.h.md)
**Used by** — [`restricted_object.cpp`](restricted_object.cpp.md) · [`space_restriction.cpp`](space_restriction.cpp.md) · [`space_restriction.h`](space_restriction.h.md) · [`space_restriction_bridge.cpp`](space_restriction_bridge.cpp.md) · [`space_restriction_bridge_inline.h`](space_restriction_bridge_inline.h.md) · [`space_restriction_composition.cpp`](space_restriction_composition.cpp.md) · [`space_restriction_composition.h`](space_restriction_composition.h.md) · [`space_restriction_holder.cpp`](space_restriction_holder.cpp.md) · [`space_restriction_manager.cpp`](space_restriction_manager.cpp.md)
**Tier floor** — T2: a declaration over an owned implementation

## Purpose

Declares the surface implemented in [`space_restriction_bridge.cpp`](space_restriction_bridge.cpp.md)
and [`space_restriction_bridge_inline.h`](space_restriction_bridge_inline.h.md).

## Exported units

- `change_implementation` — swap the geometry behind the cell.
- `object` — the current implementation, for the few callers that need its concrete face.
- `border`, `initialized`, `initialize`, `name`, `shape`, `default_restrictor`, `sphere` — delegations.
- `inside(sphere)`, `inside(vertex, partially)`, `inside(vertex, partially, radius)` — delegations.
- `on_border(position)`, `out_of_border(position)` — the two boundary tests.
- `accessible_nearest` — nearest legal vertex and point; generic over the restriction whose border and containment test the search should use.
- `accessible_neighbour_border` — forwards the cached crossable-border subset.

## Notes

Construction requires an implementation, and destruction destroys it: the cell owns what it
holds. It also carries the release timestamp both garbage collectors read, which is why it
derives from the shared time-stamped reference-counted base rather than a plain one.
