# src/xrGame/danger_cover_location_inline.h

> Construction of a cover-point danger location: every base field is set here, so the record is complete the moment it exists.

**Needs** — [`danger_cover_location.h`](danger_cover_location.h.md) · [`danger_location.h`](danger_location.h.md)
**Used by** — [`danger_cover_location.h`](danger_cover_location.h.md)
**Tier floor** — T3: field assignment

## Purpose

Holds the constructor for the cover-point danger location. It is a separate file only
because the original wanted the definition visible at every call site; a rebuild merges it
into the declaration or the implementation without loss.

## State

`Stateless.` It writes the record described in
[`danger_cover_location.cpp`](danger_cover_location.cpp.md).

## `CDangerCoverLocation` construction

**Contract** — takes the cover point (required, never absent), the level time at which the
danger was observed, the interval for which it stays relevant, the radius over which it
applies, and the squad mask of which squad members share the knowledge. The mask defaults to
all bits set, meaning "everybody knows".

**Invariants** — the base class's time, interval, radius and mask are all assigned here
rather than by a base constructor, so a danger location is fully initialised at construction
and is never observed half-built. A rebuild should keep that property; the danger set is
scanned by the brain while other code is adding to it.

**Notes** — the all-ones default mask is the permissive case, and it is the one used
whenever a danger is not squad-private. A rebuild reading the mask should treat an unset
mask as "shared", not as "nobody", or dangers will be silently invisible.
