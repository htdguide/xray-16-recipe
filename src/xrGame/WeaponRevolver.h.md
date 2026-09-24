# src/xrGame/WeaponRevolver.h

> Declares the revolver implemented in [`WeaponRevolver.cpp`](WeaponRevolver.cpp.md).

**Needs** — [`WeaponCustomPistol.h`](WeaponCustomPistol.h.md)
**Used by** — [`WeaponRevolver.cpp`](WeaponRevolver.cpp.md)
**Tier floor** — T3: a declaration only

## Purpose

Declares `CWeaponRevolver`, a semi-automatic-behaving handgun with a reload animation
chosen by the rounds remaining. Substance is in
[`WeaponRevolver.cpp`](WeaponRevolver.cpp.md).

Exported units:

- `Load` — adds the cylinder-close sound.
- `PlayAnimReload` — the six-way table indexed by rounds remaining.
- The seven other `PlayAnim*` overrides — loaded or empty rest poses.
- `PlayAnimShoot` — a distinct animation for the last round.
- `AllowFireWhileWorking` — true.
- `UpdateSounds` — also re-anchors the cylinder-close sound.
- `switch2_Reload`, `OnAnimationEnd`, `OnShot`, `net_Destroy`, `OnH_B_Chield` — empty
  overrides.

**Notes** — the class is not registered to the script virtual machine; no shipped script
names it. It is reached only through its concrete subclasses.
