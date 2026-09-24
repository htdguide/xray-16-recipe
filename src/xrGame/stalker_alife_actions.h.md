# src/xrGame/stalker_alife_actions.h

> Declares the two idle-time stalker operators: hold station with no alife simulation, and go pick up a noticed item.

**Needs** — [`stalker_base_action.h`](stalker_base_action.h.md)
**Used by** — [`stalker_alife_actions.cpp`](stalker_alife_actions.cpp.md) · [`stalker_alife_planner.cpp`](stalker_alife_planner.cpp.md) · [`stalker_planner.cpp`](stalker_planner.cpp.md)
**Tier floor** — T3: two declarations over the operator lifecycle

## Purpose

Declares the surface implemented in [`stalker_alife_actions.cpp`](stalker_alife_actions.cpp.md).

## Exported units

- `CStalkerActionNoALife` — the idle stance for a level with no alife simulation. Implements `initialize`, `execute`, `finalize`; carries the world-clock deadline after which the weapon is stowed.
- `CStalkerActionGatherItems` — walk to and take the item the memory subsystem selected. Implements `initialize`, `execute`, `finalize`; carries no state of its own.

## Notes

Both take the acting stalker and a name at construction; the name is what appears in the
planner's diagnostic output and is how a behaviour is identified when a plan is printed.
