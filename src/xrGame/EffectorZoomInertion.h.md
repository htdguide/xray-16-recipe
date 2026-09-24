# src/xrGame/EffectorZoomInertion.h

> Declares the aim-sway effector implemented in [`EffectorZoomInertion.cpp`](EffectorZoomInertion.cpp.md).

**Needs** — [`CameraEffector.h`](CameraEffector.h.md) · [`WeaponMagazined.h`](WeaponMagazined.h.md) · [`xrEngine/CameraManager.h`](../xrEngine/CameraManager.h.md)
**Used by** — [`EffectorZoomInertion.cpp`](EffectorZoomInertion.cpp.md) · [`WeaponMagazined.cpp`](WeaponMagazined.cpp.md)
**Tier floor** — T3: a declaration only

## Purpose

Declares the random-walk aim sway: the three points of the current leg, the leg clock, the
six configured parameters and the local random source. Substance in
[`EffectorZoomInertion.cpp`](EffectorZoomInertion.cpp.md).

Exported units:

- `CEffectorZoomInertion` — a permanent camera-chain effector; it suppresses itself while
  the player is turning rather than being removed.
- `Load` — read the global parameters and reset the walk.
- `Init` — re-read the parameters from a specific weapon's section, under a prefix.
- `SetParams` — push the weapon's current dispersion in; derives radius and speed.
- `ProcessCam` — advance the walk and, if the player is holding still, offset the view
  direction.
- `SetRndSeed` — seed the local random source. Unlike the recoil effector's, this one
  honours its argument.
- `CalcNextPoint`, `LoadParams` — private: draw the next leg's target; the two-level
  prefixed parameter read.
