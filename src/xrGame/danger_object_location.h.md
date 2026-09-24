# src/xrGame/danger_object_location.h

> Declares the danger location that follows a game object instead of sitting at a fixed point, implemented in [`danger_object_location.cpp`](danger_object_location.cpp.md).

**Needs** — [`danger_location.h`](danger_location.h.md) · [`danger_object_location_inline.h`](danger_object_location_inline.h.md)
**Used by** — [`agent_explosive_manager.cpp`](agent_explosive_manager.cpp.md) · [`danger_object_location.cpp`](danger_object_location.cpp.md) · [`danger_object_location_inline.h`](danger_object_location_inline.h.md)
**Tier floor** — T3: a declaration

## Purpose

Declares `CDangerObjectLocation`, the only shipped implementation of
[`CDangerLocation`](danger_location.h.md) in this directory: a squad-level warning attached
to a particular game object, so that the warned area tracks the object as it moves.
Substance in [`danger_object_location.cpp`](danger_object_location.cpp.md).

Exported units:

- `CDangerObjectLocation` — construction binds the object, the timestamp, the interval, the
  radius and the squad mask; the mask defaults to *every member*.
- `position` — the bound object's current position, re-read on each call.
- `useful` — always true; this location does not expire on the clock.
- match against a game object — true when the identifiers agree.
