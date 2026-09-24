# src/xrGame/stalker_alife_task_actions.h

> Declares the two operators that execute an alife assignment: hold station, and travel to the smart terrain's job.

**Needs** — [`stalker_base_action.h`](stalker_base_action.h.md)
**Used by** — [`stalker_alife_planner.cpp`](stalker_alife_planner.cpp.md) · [`stalker_alife_task_actions.cpp`](stalker_alife_task_actions.cpp.md)
**Tier floor** — T3: two declarations over the operator lifecycle

## Purpose

Declares the surface implemented in [`stalker_alife_task_actions.cpp`](stalker_alife_task_actions.cpp.md).

## Exported units

- `CStalkerActionSolveZonePuzzle` — the idle stance when the alife simulation is running and no job is pending. Implements `initialize`, `execute`, `finalize`; carries the world-clock deadline after which the weapon is stowed.
- `CStalkerActionSmartTerrain` — travel to the assigned job location, coarse leg then fine leg. Implements `initialize`, `execute`, `finalize`; carries no state.

## Notes

Both names are vestigial. No zone puzzle exists; the first is simply the idle operator, and
the property it achieves is a never-satisfiable placeholder that keeps the idle brain
running. See [`stalker_alife_planner.cpp`](stalker_alife_planner.cpp.md).
