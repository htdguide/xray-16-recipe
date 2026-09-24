# src/xrGame/GrenadeLauncher.h

> Declares the under-barrel grenade launcher attachment, implemented in [`GrenadeLauncher.cpp`](GrenadeLauncher.cpp.md).

**Needs** — [`inventory_item_object.h`](inventory_item_object.h.md)
**Used by** — [`GrenadeLauncher.cpp`](GrenadeLauncher.cpp.md) · [`Scope.cpp`](Scope.cpp.md) · [`WeaponMagazined.cpp`](WeaponMagazined.cpp.md) · [`WeaponMagazinedWGrenade.cpp`](WeaponMagazinedWGrenade.cpp.md)
**Tier floor** — T3: a declaration only

## Purpose

Declares `CGrenadeLauncher`, a generic inventory item that carries one tuned number for
the weapon it attaches to. Substance — such as it is — is in
[`GrenadeLauncher.cpp`](GrenadeLauncher.cpp.md).

Exported units:

- `CGrenadeLauncher` — the attachment item.
- `Load` — read the muzzle velocity from the section.
- `GetGrenadeVel` — hand it to the weapon.
- `net_Spawn`, `net_Destroy`, `UpdateCL`, `OnH_A_Chield`, `OnH_B_Independent` —
  lifecycle overrides that add nothing.
