# src/xrGame/stalker_low_cover_actions.h

> Declares the three actions of fighting from cover that only protects you crouched.

**Needs** — [`stalker_low_cover_actions.cpp`](stalker_low_cover_actions.cpp.md) · [`stalker_combat_action_base.h`](stalker_combat_action_base.h.md) · [`cover_point.h`](cover_point.h.md)
**Used by** — [`stalker_low_cover_actions.cpp`](stalker_low_cover_actions.cpp.md) · [`stalker_low_cover_planner.cpp`](stalker_low_cover_planner.cpp.md)
**Tier floor** — T2: three action objects per creature.

## Purpose

Declares the surface implemented in
[`stalker_low_cover_actions.cpp`](stalker_low_cover_actions.cpp.md).

## Exported units

- `GetReadyToKillLowCover` — duck behind the cover and bring the weapon up.
- `KillEnemyLowCover` — stand up and shoot over it.
- `HoldPositionLowCover` — stand up and watch, firing only if the squad needs it.

The last two each declare a timestamp field and a private random source that no code in the
file reads. They are the residue of a posture-alternation the shipped behaviour does not
have; a rebuild should not carry them over.
