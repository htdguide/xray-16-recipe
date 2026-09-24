# src/xrGame/NoGravityZone.h

> Declares the weightlessness anomaly implemented in [`NoGravityZone.cpp`](NoGravityZone.cpp.md).

**Needs** — [`CustomZone.h`](CustomZone.h.md) · [`NoGravityZone.cpp`](NoGravityZone.cpp.md)
**Used by** — [`HairsZone_script.cpp`](HairsZone_script.cpp.md) · [`NoGravityZone.cpp`](NoGravityZone.cpp.md)
**Tier floor** — T3: a declaration only

## Purpose

Declares `CNoGravityZone`, the anomaly that clears gravity for everything inside it. Substance
is in [`NoGravityZone.cpp`](NoGravityZone.cpp.md).

Exported units:

- `CNoGravityZone` — the anomaly. It carries no state of its own; everything it changes lives
  on the objects it touches.
- `enter_Zone` / `exit_Zone` — clear and restore gravity, with the restore deliberately running
  before the base class's exit.
- `UpdateWorkload` — re-asserts weightlessness on everything inside, every tick.
- `switchGravity` (private) — the one operation, with separate paths for a rigid body and for a
  creature's character controller.

This class is exported to the script layer from
[`HairsZone_script.cpp`](HairsZone_script.cpp.md), not from a sibling of its own.
