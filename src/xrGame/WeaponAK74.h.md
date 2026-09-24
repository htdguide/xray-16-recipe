# src/xrGame/WeaponAK74.h

> Declares the standard assault rifle implemented in [`WeaponAK74.cpp`](WeaponAK74.cpp.md).

**Needs** — [`WeaponMagazinedWGrenade.h`](WeaponMagazinedWGrenade.h.md)
**Used by** — [`WeaponAK74.cpp`](WeaponAK74.cpp.md)
**Tier floor** — T3: a declaration only

## Purpose

Declares `CWeaponAK74`, a grenade-launcher-capable magazined weapon with no behaviour of
its own. Substance — such as it is — is in [`WeaponAK74.cpp`](WeaponAK74.cpp.md).

Exported units:

- `CWeaponAK74(sound_type)` — defaults to the submachine-gun sound class; registered to
  the script virtual machine under its own name.
