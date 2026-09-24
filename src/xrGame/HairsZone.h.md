# src/xrGame/HairsZone.h

> Declares the motion-triggered anomaly implemented in [`HairsZone.cpp`](HairsZone.cpp.md).

**Needs** — [`CustomZone.h`](CustomZone.h.md) · [`ZoneVisual.h`](ZoneVisual.h.md) · [`Include/xrRender/KinematicsAnimated.h`](../Include/xrRender/KinematicsAnimated.h.md)
**Used by** — [`HairsZone.cpp`](HairsZone.cpp.md) · [`HairsZone_script.cpp`](HairsZone_script.cpp.md)
**Tier floor** — T3: a declaration only

## Purpose

Declares `CHairsZone`, a visual anomaly that wakes on occupant speed. Substance is in
[`HairsZone.cpp`](HairsZone.cpp.md); the Lua export is in
[`HairsZone_script.cpp`](HairsZone_script.cpp.md).

Exported units:

- `CHairsZone` — a visual zone carrying one tuned number, the speed that wakes it.
- `Load` — reads that number.
- `Affect` — the randomly-directed upward hit.
- `CheckForAwaking` — the motion trigger.
- `BlowoutState` — the blowout tick.
- the script-registration entry point, defined in the script sibling.
