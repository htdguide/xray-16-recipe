# src/xrGame/WeaponPistol.h

> Declares the pistol implemented in [`WeaponPistol.cpp`](WeaponPistol.cpp.md).

**Needs** — [`WeaponCustomPistol.h`](WeaponCustomPistol.h.md)
**Used by** — [`WeaponFORT.h`](WeaponFORT.h.md) · [`WeaponHPSA.cpp`](WeaponHPSA.cpp.md) · [`WeaponHPSA.h`](WeaponHPSA.h.md) · [`WeaponPM.cpp`](WeaponPM.cpp.md) · [`WeaponPM.h`](WeaponPM.h.md) · [`WeaponPistol.cpp`](WeaponPistol.cpp.md) · [`WeaponUSP45.h`](WeaponUSP45.h.md) · [`WeaponWalther.h`](WeaponWalther.h.md)
**Tier floor** — T3: a declaration only

## Purpose

Declares `CWeaponPistol`, a semi-automatic weapon with a slide that locks back when
empty. Substance is in [`WeaponPistol.cpp`](WeaponPistol.cpp.md).

Exported units:

- `Load` — adds the slide-close sound to the inherited sound set.
- The seven `PlayAnim*` overrides — each selects a loaded or empty variant.
- `PlayAnimShoot` — a distinct animation for the last round in the magazine.
- `AllowFireWhileWorking` — true; a pistol re-triggers while its shot animation runs.
- `UpdateSounds` — also re-anchors the slide-close sound.
- `switch2_Reload`, `OnAnimationEnd`, `net_Destroy`, `OnH_B_Chield` — empty overrides.
