# src/xrGame/danger_cover_location.cpp

> A danger location whose position is borrowed from a cover point rather than stored: the place a creature was shot at from, remembered as "that corner".

**Needs** — [`danger_cover_location.h`](danger_cover_location.h.md) · [`cover_point.h`](cover_point.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: one indirection

## Purpose

One of the danger-location kinds. Where the base kind carries a coordinate, this one carries
a reference to a cover point and reports that point's position.

The distinction is not a memory optimisation. A cover point is a *named place in the
navigation topology* — the pathfinder and the cover search both reason about it — so a
danger recorded against a cover point can be matched against the cover the creature is about
to move to. A danger recorded as a bare coordinate can only be compared by distance. This is
how a creature declines to take the cover it just saw a grenade land behind.

## State

```text
RECORD DangerCoverLocation EXTENDS DangerLocation
  cover : CoverPoint        # invariant: never absent; borrowed, never owned
```

The cover point is borrowed. Its lifetime is the level's, held by the cover index, and it
outlives any danger record — which is why the reference is safe to hold and why nothing here
releases it.

## `position`

**Contract** — the referenced cover point's position. Pure delegation; no copy is kept, so
the record automatically follows a cover point that is re-derived.

## `CDangerCoverLocation` construction

**Contract** — declared here, defined in
[`danger_cover_location_inline.h`](danger_cover_location_inline.h.md). Takes the cover
point, the level time the danger was recorded, the interval it stays live, the radius it
covers and the squad mask of who knows about it.
