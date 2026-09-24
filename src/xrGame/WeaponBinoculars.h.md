# src/xrGame/WeaponBinoculars.h

> Declares the binoculars implemented in [`WeaponBinoculars.cpp`](WeaponBinoculars.cpp.md).

**Needs** — [`WeaponCustomPistol.h`](WeaponCustomPistol.h.md) · [`WeaponBinocularsVision.h`](WeaponBinocularsVision.h.md)
**Used by** — [`WeaponBinoculars.cpp`](WeaponBinoculars.cpp.md) · [`WeaponScript.cpp`](WeaponScript.cpp.md)
**Tier floor** — T3: a declaration only

## Purpose

Declares `CWeaponBinoculars`, a weapon that cannot fire and whose trigger zooms.
Substance is in [`WeaponBinoculars.cpp`](WeaponBinoculars.cpp.md).

Exported units:

- `Load` — zoom sounds, and whether the target brackets are present.
- `Action` — rewrites the fire binding to the zoom binding.
- `OnZoomIn` / `OnZoomOut` — sounds, and the bracket overlay's lifetime.
- `ZoomInc` / `ZoomDec` — stepped magnification.
- `UpdateCL` / `render_item_ui` / `render_item_ui_query` — drive and draw the overlay.
- `can_kill` — always false; `use_crosshair` — always false.
- `GetBriefInfo` — name and icon only.
- `save` / `load` — the remembered magnification.
- `net_Relcase` — sever the overlay's references to a dying object.

The free function `GetZoomData(narrowest_fov, out step, out widest_fov)` is defined in
the implementation and declared in [`Weapon.cpp`](Weapon.cpp.md); it is the shared
stepped-zoom rule for every dynamic optic, and lives here only by accident.
