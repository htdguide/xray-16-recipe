# src/xrGame/stalker_get_distance_actions.h

> Declares the two actions of breaking off a fight you cannot win at this range: run to cover, then wait there.

**Needs** — [`stalker_get_distance_actions.cpp`](stalker_get_distance_actions.cpp.md) · [`stalker_combat_actions.h`](stalker_combat_actions.h.md)
**Used by** — [`stalker_get_distance_actions.cpp`](stalker_get_distance_actions.cpp.md) · [`stalker_get_distance_planner.cpp`](stalker_get_distance_planner.cpp.md)
**Tier floor** — T2: two action objects per creature.

## Purpose

Declares the surface implemented in
[`stalker_get_distance_actions.cpp`](stalker_get_distance_actions.cpp.md).

## Exported units

- `RunToCover` — sprint to a position closer to the enemy with something between, firing on
  the way if the enemy is visible.
- `WaitInCover` — hold there for one to three seconds, then give up the hold.
