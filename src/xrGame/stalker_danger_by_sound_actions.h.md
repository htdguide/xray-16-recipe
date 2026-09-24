# src/xrGame/stalker_danger_by_sound_actions.h

> Declares five actions for reacting to a suspicious sound. None of them is ever reached, and all five have the same body.

**Needs** — [`stalker_danger_by_sound_actions.cpp`](stalker_danger_by_sound_actions.cpp.md) · [`stalker_combat_actions.h`](stalker_combat_actions.h.md)
**Used by** — [`stalker_danger_by_sound_actions.cpp`](stalker_danger_by_sound_actions.cpp.md) · [`stalker_danger_by_sound_planner.cpp`](stalker_danger_by_sound_planner.cpp.md)
**Tier floor** — T2: five action objects per creature, four of which are never instantiated.

## Purpose

Declares the surface implemented in
[`stalker_danger_by_sound_actions.cpp`](stalker_danger_by_sound_actions.cpp.md). Read that
twin first: this branch of the brain is unfinished, and the honest summary of what it does
belongs there.

## Exported units

- `DangerBySoundListenTo` — stop and listen.
- `DangerBySoundCheck` — go and check.
- `DangerBySoundTakeCover` — get behind something.
- `DangerBySoundLookOut` — lean out and look.
- `DangerBySoundLookAround` — sweep.

The names describe an intended behaviour. The implementations do not distinguish between
them.
