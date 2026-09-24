# src/xrGame/stalker_search_actions.h

> Declares the three actions of the lost-enemy search.

**Needs** — [`stalker_search_actions.cpp`](stalker_search_actions.cpp.md) · [`stalker_combat_action_base.h`](stalker_combat_action_base.h.md)
**Used by** — [`stalker_search_actions.cpp`](stalker_search_actions.cpp.md) · [`stalker_search_planner.cpp`](stalker_search_planner.cpp.md)
**Tier floor** — T2: three action types over the combat action base.

## Purpose

Declares the surface implemented in
[`stalker_search_actions.cpp`](stalker_search_actions.cpp.md). All three derive from the
combat action base, so each inherits combat's sound barks, weapon handling and update
cadence and adds only its own movement and sight decisions.

All three carry an identical pair of fields: a reference to the **combat** planner's world
state (written when the creature is hit, to force the parent to re-plan) and the hit-clock
watermark taken at initialize. The duplication is real — the three could share a base
carrying the hit interrupt — and a rebuild should fold it into one.

## Exported units

- `ReachEnemyLocation` — walk to the enemy's last remembered position; sets
  `EnemyLocationReached` on arrival.
- `ReachAmbushLocation` — find and occupy a cover point watching that position; sets
  `AmbushLocationReached` on arrival.
- `HoldAmbushLocation` — crouch and watch; after both its inertia window and the memory's
  one-minute horizon expire, disables the enemy in memory, which is what ends the search.

Each exposes the same three lifecycle operations — `initialize`, `execute`, `finalize` — and
the order is load-bearing: initialize fixes posture and snapshots the hit clock before the
first execute runs.
