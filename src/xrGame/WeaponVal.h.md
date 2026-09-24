# src/xrGame/WeaponVal.h

> Declares the VAL rifle leaf implemented in [`WeaponVal.cpp`](WeaponVal.cpp.md).

**Needs** — [`WeaponMagazined.h`](WeaponMagazined.h.md) · [Seam: Script binding layer](../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer)
**Used by** — [`WeaponVal.cpp`](WeaponVal.cpp.md)
**Tier floor** — T3: a declaration only

## Purpose

Names `CWeaponVal` so the class-identifier factory can construct it and so the script
layer can register it. It also declares that its script registration is *inherited* from
the magazine-fed weapon base rather than adding an exported surface of its own — the
weapon is visible to scripts as a magazine-fed weapon.

Exported units:

- `CWeaponVal` — a magazine-fed weapon leaf; substance in
  [`WeaponVal.cpp`](WeaponVal.cpp.md).
