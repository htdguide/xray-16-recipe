# src/xrGame/WeaponCustomPistol.h

> Declares the semi-automatic firing behaviour implemented in [`WeaponCustomPistol.cpp`](WeaponCustomPistol.cpp.md).

**Needs** — [`WeaponMagazined.h`](WeaponMagazined.h.md)
**Used by** — [`WeaponBinoculars.cpp`](WeaponBinoculars.cpp.md) · [`WeaponBinoculars.h`](WeaponBinoculars.h.md) · [`WeaponCustomPistol.cpp`](WeaponCustomPistol.cpp.md) · [`WeaponKnife.h`](WeaponKnife.h.md) · [`WeaponPistol.cpp`](WeaponPistol.cpp.md) · [`WeaponPistol.h`](WeaponPistol.h.md) · [`WeaponRPG7.cpp`](WeaponRPG7.cpp.md) · [`WeaponRPG7.h`](WeaponRPG7.h.md) · [`WeaponRevolver.cpp`](WeaponRevolver.cpp.md) · [`WeaponRevolver.h`](WeaponRevolver.h.md) · [`WeaponSVD.cpp`](WeaponSVD.cpp.md) · [`WeaponSVD.h`](WeaponSVD.h.md) · [`WeaponSVU.h`](WeaponSVU.h.md) · [`WeaponShotgun.cpp`](WeaponShotgun.cpp.md) · _and 1 more_
**Tier floor** — T3: a declaration only

## Purpose

Declares `CWeaponCustomPistol`: one round per trigger pull, with the release deferred
until the weapon has cycled. Substance is in
[`WeaponCustomPistol.cpp`](WeaponCustomPistol.cpp.md). It is the parent of every pistol
(through [`WeaponPistol.h`](WeaponPistol.h.md)) and of the two semi-automatic sniper
rifles directly.

Exported units:

- `CWeaponCustomPistol()` — a magazined weapon tagged with the pistol sound class, which
  is what the AI's hearing system uses to judge a shot's character.
- `switch2_Fire` — arm one round without setting the firing flag.
- `FireEnd` — ignore the trigger release until the shot clock has expired.
- `GetCurrentFireMode` — always 1; no selector.
