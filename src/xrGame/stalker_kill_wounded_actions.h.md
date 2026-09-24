# src/xrGame/stalker_kill_wounded_actions.h

> Declares the five steps of finishing a downed enemy: walk over, aim, say something, shoot, pause.

**Needs** — [`stalker_kill_wounded_actions.cpp`](stalker_kill_wounded_actions.cpp.md) · [`stalker_combat_actions.h`](stalker_combat_actions.h.md)
**Used by** — [`stalker_kill_wounded_actions.cpp`](stalker_kill_wounded_actions.cpp.md) · [`stalker_kill_wounded_planner.cpp`](stalker_kill_wounded_planner.cpp.md)
**Tier floor** — T2: five action objects per creature; one of them sends a damage message.

## Purpose

Declares the surface implemented in
[`stalker_kill_wounded_actions.cpp`](stalker_kill_wounded_actions.cpp.md).

## Exported units

- `ReachWounded` — walk to the downed enemy.
- `AimWounded` — settle the head on it.
- `PrepareWounded` — deliver the execution line and wait for it to finish.
- `KillWounded` — fire, and guarantee the kill even when the shot cannot land.
- `PauseAfterKill` — a second of standing over the body.
