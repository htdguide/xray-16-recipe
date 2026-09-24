# src/xrGame/WeaponVintorez.h

> Declares the Vintorez rifle leaf implemented in [`WeaponVintorez.cpp`](WeaponVintorez.cpp.md).

**Needs** — [`WeaponMagazined.h`](WeaponMagazined.h.md) · [Seam: Script binding layer](../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer)
**Used by** — [`WeaponVintorez.cpp`](WeaponVintorez.cpp.md)
**Tier floor** — T3: a declaration only

## Purpose

Names `CWeaponVintorez` for the class-identifier factory, and declares that its script
registration is the magazine-fed weapon base's — scripts see it as a magazine-fed weapon,
not as a distinct exported type.

Exported units:

- `CWeaponVintorez` — a magazine-fed weapon leaf; substance in
  [`WeaponVintorez.cpp`](WeaponVintorez.cpp.md).
