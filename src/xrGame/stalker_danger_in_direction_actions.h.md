# src/xrGame/stalker_danger_in_direction_actions.h

> Declares the five steps of reacting to a threat you can face: cover, look out, hold, flank, search.

**Needs** — [`stalker_danger_in_direction_actions.cpp`](stalker_danger_in_direction_actions.cpp.md) · [`stalker_combat_actions.h`](stalker_combat_actions.h.md)
**Used by** — [`stalker_danger_in_direction_actions.cpp`](stalker_danger_in_direction_actions.cpp.md) · [`stalker_danger_in_direction_planner.cpp`](stalker_danger_in_direction_planner.cpp.md)
**Tier floor** — T2: five action objects per creature, each searching the cover database.

## Purpose

Declares the surface implemented in
[`stalker_danger_in_direction_actions.cpp`](stalker_danger_in_direction_actions.cpp.md).

## Exported units

- `DangerInDirectionTakeCover` — get behind something facing the threat. Carries an unused
  sight-preference flag; see the implementation twin.
- `DangerInDirectionLookOut` — move to a position with a line on the threat. Carries its own
  random source, seeded from the processor's cycle counter, deciding crouch or stand.
- `DangerInDirectionHoldPosition` — wait there with the weapon up.
- `DangerInDirectionDetour` — work around to a different angle.
- `DangerInDirectionSearch` — move to an ambush position and then stop believing in the
  threat.
