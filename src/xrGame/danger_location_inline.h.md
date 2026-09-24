# src/xrGame/danger_location_inline.h

> Position matching by proximity, the base refusal to match an entity, and the mask accessor.

**Needs** — [`danger_location.h`](danger_location.h.md)
**Used by** — [`danger_location.h`](danger_location.h.md)
**Tier floor** — T3: comparisons

## Purpose

Bodies for three of the declarations in [`danger_location.h`](danger_location.h.md), split
out for inlining. The contracts are stated there; only two things are worth restating.

## State

Adds nothing.

## Notes

- Position matching is *approximate* — the math layer's similarity test, not equality. Two
  danger locations a centimetre apart are one location. This is what keeps a squad from
  accumulating a warning per frame for a continuously dangerous spot.
- The base match against an entity returns false unconditionally, including for an absent
  entity. An implementor that binds to an object must override it; one that does not is
  declaring itself a pure place, which the object-destroyed sweep then correctly leaves
  alone.
