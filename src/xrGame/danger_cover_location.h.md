# src/xrGame/danger_cover_location.h

> Declares the danger location that names a cover point, implemented in [`danger_cover_location.cpp`](danger_cover_location.cpp.md) and [`danger_cover_location_inline.h`](danger_cover_location_inline.h.md).

**Needs** — [`danger_location.h`](danger_location.h.md) · [`danger_cover_location_inline.h`](danger_cover_location_inline.h.md) · [`cover_point.h`](cover_point.h.md)
**Used by** — [`danger_cover_location.cpp`](danger_cover_location.cpp.md) · [`danger_cover_location_inline.h`](danger_cover_location_inline.h.md) · [`stalker_combat_actions.cpp`](stalker_combat_actions.cpp.md) · [`stalker_danger_unknown_actions.cpp`](stalker_danger_unknown_actions.cpp.md)
**Tier floor** — T3: a declaration

## Purpose

Declares the danger-location kind whose position comes from a cover point instead of a
stored coordinate. Substance is in
[`danger_cover_location.cpp`](danger_cover_location.cpp.md).

Exported units:

- `CDangerCoverLocation` — the record: a borrowed cover point over the base location's time,
  interval, radius and squad mask.
- `position` — the borrowed point's position, overriding the base.
