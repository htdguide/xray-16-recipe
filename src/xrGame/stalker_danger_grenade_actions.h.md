# src/xrGame/stalker_danger_grenade_actions.h

> Declares the five steps of surviving a grenade: run, crouch, run again, sweep, stop.

**Needs** — [`stalker_danger_grenade_actions.cpp`](stalker_danger_grenade_actions.cpp.md) · [`stalker_combat_actions.h`](stalker_combat_actions.h.md)
**Used by** — [`stalker_danger_grenade_actions.cpp`](stalker_danger_grenade_actions.cpp.md) · [`stalker_danger_grenade_planner.cpp`](stalker_danger_grenade_planner.cpp.md)
**Tier floor** — T2: five action objects per creature.

## Purpose

Declares the surface implemented in
[`stalker_danger_grenade_actions.cpp`](stalker_danger_grenade_actions.cpp.md). All five
derive from the combat action base for its sound masking and cover helper.

## Exported units

- `DangerGrenadeTakeCover` — get away from the grenade before it goes off.
- `DangerGrenadeWaitForExplosion` — crouch behind cover and wait it out.
- `DangerGrenadeTakeCoverAfterExplosion` — reposition once the blast is past. Carries the
  same coin-flipped sight flag as the unknown-danger version.
- `DangerGrenadeLookAround` — crouch and sweep, weapon up.
- `DangerGrenadeSearch` — the branch terminator: stand still and stop reacting.
