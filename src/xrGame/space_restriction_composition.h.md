# src/xrGame/space_restriction_composition.h

> Declares the union-of-restrictors volume, which doubles as the placeholder for a named restrictor that has not spawned.

**Needs** — [`space_restriction_base.h`](space_restriction_base.h.md) · [`space_restriction_bridge.h`](space_restriction_bridge.h.md) · [`space_restriction_holder.h`](space_restriction_holder.h.md) · [`space_restriction_composition_inline.h`](space_restriction_composition_inline.h.md)
**Used by** — [`space_restriction_composition.cpp`](space_restriction_composition.cpp.md) · [`space_restriction_composition_inline.h`](space_restriction_composition_inline.h.md) · [`space_restriction_holder.cpp`](space_restriction_holder.cpp.md)
**Tier floor** — T2: a declaration over a member list and a bounding sphere

## Purpose

Declares the surface implemented in [`space_restriction_composition.cpp`](space_restriction_composition.cpp.md)
and [`space_restriction_composition_inline.h`](space_restriction_composition_inline.h.md).

## Exported units

- `initialize` — resolve members, merge borders, compute the enclosing sphere.
- `inside(sphere)` — union containment behind a sphere reject. The per-vertex forms are inherited.
- `name` — the normalized member list, which is also the registry key.
- `shape` — always false, which makes compositions eligible for garbage collection.
- `default_restrictor` — always false; a composition is never one of the level's defaults.
- `sphere` — a hard failure; compositions do not nest.
- `test_correctness` — checked-build border connectivity check.

## Notes

The live-composition counter declared here is a leak tripwire; nothing in the shipped build
reads it.
