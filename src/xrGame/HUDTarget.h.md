# src/xrGame/HUDTarget.h

> Declares the look-at ray and the cursor drawn where it lands, implemented in [`HUDTarget.cpp`](HUDTarget.cpp.md).

**Needs** — [`HUDCrosshair.h`](HUDCrosshair.h.md) · [`xrCDB/xr_collide_defs.h`](../xrCDB/xr_collide_defs.h.md)
**Used by** — [`HUDManager.cpp`](HUDManager.cpp.md) · [`HUDTarget.cpp`](HUDTarget.cpp.md)
**Tier floor** — T3: a declaration only

## Purpose

Declares `CHUDTarget`, which owns the reticle and the per-frame look-at ray, and the
small record that carries a ray cast's accumulated state through the acceptance callback.
Substance is in [`HUDTarget.cpp`](HUDTarget.cpp.md).

Exported units:

- `SPickParam` — the ray's accumulator: the accepted hit, the running visibility product
  that lets the ray see through transparent materials, and the surface count kept for
  the debug readout.
- `CHUDTarget` — the module.
- `CursorOnFrame` — cast the ray and record what it found.
- `Render` — draw the cursor or the reticle, tinted and captioned by the target.
- `Load` — load the reticle's configuration.
- `GetRQ` — the last pick result. Read by the depth-of-field effector, which focuses on
  whatever the player is looking at, and by anything asking what is under the crosshair.
- `GetRQVis` — the accumulated visibility of that hit.
- `GetHUDCrosshair` — the reticle, so a weapon can push its dispersion into it.
- `ShowCrosshair` — reticle versus dot cursor.
- `net_Relcase` — drop a reference to an object being destroyed.
