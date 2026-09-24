# src/xrGame/HUDCrosshair.h

> Declares the dispersion-driven aiming reticle, implemented in [`HUDCrosshair.cpp`](HUDCrosshair.cpp.md).

**Needs** — [`xrUICore/ui_defs.h`](../xrUICore/ui_defs.h.md)
**Used by** — [`HUDCrosshair.cpp`](HUDCrosshair.cpp.md) · [`HUDTarget.cpp`](HUDTarget.cpp.md) · [`HUDTarget.h`](HUDTarget.h.md)
**Tier floor** — T3: a declaration only

## Purpose

Declares `CHUDCrosshair` and names the one configuration section its sizes come from.
Substance is in [`HUDCrosshair.cpp`](HUDCrosshair.cpp.md).

Exported units:

- `CHUDCrosshair` — the reticle.
- `Load` — read tick length, minimum and maximum radius, and default colour from the
  fixed cursor section.
- `SetDispersion` — project a dispersion half-angle into a screen radius.
- `OnRender` — draw the four ticks and the centre mark.
- `cross_color` — a public field, written each frame by the targeting module so the
  reticle can take its colour from what it is pointing at.
- `SetFirstBulletDispertion`, `OnRenderFirstBulletDispertion` — the debug-only second
  reticle showing the cold-weapon cone.
- `HUD_CURSOR_SECTION` — the configuration section name, frozen by the shipped data.
