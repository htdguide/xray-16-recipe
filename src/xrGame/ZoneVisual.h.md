# src/xrGame/ZoneVisual.h

> Declares the animated anomaly implemented in [`ZoneVisual.cpp`](ZoneVisual.cpp.md).

**Needs** — [`CustomZone.h`](CustomZone.h.md)
**Used by** — [`AmebaZone.cpp`](AmebaZone.cpp.md) · [`AmebaZone.h`](AmebaZone.h.md) · [`HairsZone.cpp`](HairsZone.cpp.md) · [`HairsZone.h`](HairsZone.h.md) · [`ZoneVisual.cpp`](ZoneVisual.cpp.md)
**Tier floor** — T3: a declaration only

## Purpose

Declares `CVisualZone`, an anomaly zone with a skinned model, and the state that binds it:
two resolved motion handles and the blowout-relative window during which the attack motion
plays. Substance is in [`ZoneVisual.cpp`](ZoneVisual.cpp.md).

Exported units:

- `net_Spawn` — resolve both motions against the model, start the idle one, become visible.
- `Load` — read the attack window from the configuration section.
- `SwitchZoneState` — restore the idle motion when leaving the blowout state.
- `UpdateBlowout` — fire the attack and idle motions at the window's edges.

**Notes** — this header declares its class without including the base's header, relying on
the including file to have done so; that is an incidental build-ordering habit, not a
design decision.
