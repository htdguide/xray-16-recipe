# src/xrGame/alife_smart_terrain_registry_inline.h

> The two read accessors of the smart-terrain index.

**Needs** — [`alife_smart_terrain_registry.h`](alife_smart_terrain_registry.h.md)
**Used by** — [`alife_smart_terrain_registry.h`](alife_smart_terrain_registry.h.md)
**Tier floor** — T3: a map lookup

## Purpose

Bodies for the index's read accessors. Substance is in
[`alife_smart_terrain_registry.cpp`](alife_smart_terrain_registry.cpp.md).

## `object` and `objects`

**Contract** — `object` resolves an entity identifier to a smart terrain and **requires it
to be present**; there is no tolerant form and no absent result. `objects` yields the
whole index for a caller that must sweep it.

**Invariants** — the strictness is the point. Every caller here already holds an
identifier it obtained *from* a smart terrain relationship — a creature's assigned
terrain, a job's owner — so absence means the relationship outlived the terrain, which is
a registration bug and not a condition to recover from. A rebuild that softens this to an
optional result will convert a diagnosable fault into a creature that silently does
nothing.
