# src/xrGame/stalker_anomaly_actions.h

> Declares the two things a stalker does about an anomaly: leave the one it is standing in, and probe for the one it suspects.

**Needs** — [`stalker_anomaly_actions.cpp`](stalker_anomaly_actions.cpp.md) · [`stalker_base_action.h`](stalker_base_action.h.md)
**Used by** — [`stalker_anomaly_actions.cpp`](stalker_anomaly_actions.cpp.md) · [`stalker_anomaly_planner.cpp`](stalker_anomaly_planner.cpp.md)
**Tier floor** — T2: two action objects per creature.

## Purpose

Declares the surface implemented in
[`stalker_anomaly_actions.cpp`](stalker_anomaly_actions.cpp.md).

## Exported units

- `GetOutOfAnomaly` — walk out of the zone the creature is inside. Carries two scratch
  lists of entity identifiers used to hand the movement restrictor set a diff; they are
  fields rather than locals only so that the per-frame update does not allocate, which is
  an optimization a rebuild may ignore.
- `DetectAnomaly` — throw a bolt to find out whether the suspected anomaly is real.

Both expose the standard action lifecycle: entry, per-cycle execution, exit.
