# src/xrGame/AmebaZone.h

> Declares the slowing, upward-throwing anomaly implemented in [`AmebaZone.cpp`](AmebaZone.cpp.md).

**Needs** — [`ZoneVisual.h`](ZoneVisual.h.md) · [`CustomZone.h`](CustomZone.h.md) · [`xrPhysics/PHUpdateObject.h`](../xrPhysics/PHUpdateObject.h.md) · [`Include/xrRender/KinematicsAnimated.h`](../Include/xrRender/KinematicsAnimated.h.md)
**Used by** — [`AmebaZone.cpp`](AmebaZone.cpp.md) · [`HairsZone_script.cpp`](HairsZone_script.cpp.md)
**Tier floor** — T3: a declaration

## Purpose

Declares `CAmebaZone`. Substance is in [`AmebaZone.cpp`](AmebaZone.cpp.md).

Its load-bearing content is the **double inheritance**: the anomaly is both a visual zone —
an anomaly with an animated model — and a *physics-step participant*. The second is what
lets it impose a continuous speed limit on creatures inside, which an ordinary anomaly
cannot do.

Exported units:

- `CAmebaZone` — the anomaly. Holds one tuning value, the in-zone speed limit.
- `Load` — reads that value.
- `Affect` — the per-object upward throw.
- `PhTune` — the in-physics-step speed clamp. Its sibling, the post-step data update, is
  deliberately empty: this anomaly reads nothing back from the solve.
- `BlowoutState` — advance the discharge and apply it to everything inside, every update.
- `SwitchZoneState` — join and leave the physics world's participant list on the discharge
  edges.
- `distance_to_center` — see the implementation note in
  [`AmebaZone.cpp`](AmebaZone.cpp.md); the shipped version measures only one axis.
