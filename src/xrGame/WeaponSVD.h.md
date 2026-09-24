# src/xrGame/WeaponSVD.h

> Declares the semi-automatic sniper rifle implemented in [`WeaponSVD.cpp`](WeaponSVD.cpp.md).

**Needs** — [`WeaponCustomPistol.h`](WeaponCustomPistol.h.md)
**Used by** — [`WeaponSVD.cpp`](WeaponSVD.cpp.md)
**Tier floor** — T3: a declaration only

## Purpose

Declares `CWeaponSVD`, a semi-automatic weapon that additionally locks itself for the
length of its shot animation. Substance is in [`WeaponSVD.cpp`](WeaponSVD.cpp.md).

Exported units:

- `switch2_Fire` — arm one round and mark the weapon pending.
- `OnAnimationEnd` — clear pending when the fire animation completes.
