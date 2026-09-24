# src/xrGame/stalker_danger_unknown_actions.h

> Declares the three steps of reacting to a threat you cannot locate: get behind something, sweep, then remember the place is dangerous.

**Needs** — [`stalker_danger_unknown_actions.cpp`](stalker_danger_unknown_actions.cpp.md) · [`stalker_combat_actions.h`](stalker_combat_actions.h.md)
**Used by** — [`stalker_danger_unknown_actions.cpp`](stalker_danger_unknown_actions.cpp.md) · [`stalker_danger_unknown_planner.cpp`](stalker_danger_unknown_planner.cpp.md)
**Tier floor** — T2: three action objects per creature.

## Purpose

Declares the surface implemented in
[`stalker_danger_unknown_actions.cpp`](stalker_danger_unknown_actions.cpp.md). All three
derive from the combat action base rather than the plain one, which gives them the sound
masking and the cover-point helper; they never fire.

## Exported units

- `DangerUnknownTakeCover` — run to cover. Carries one flag, decided by a coin flip at
  entry, choosing whether to watch the way it is running or watch the cover ahead.
- `DangerUnknownLookAround` — crouch and sweep.
- `DangerUnknownSearch` — the terminating step: publish the place as dangerous and let the
  branch end.
