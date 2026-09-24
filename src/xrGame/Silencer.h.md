# src/xrGame/Silencer.h

> Declares the silencer attachment implemented in [`Silencer.cpp`](Silencer.cpp.md).

**Needs** — [`inventory_item_object.h`](inventory_item_object.h.md)
**Used by** — [`Scope.cpp`](Scope.cpp.md) · [`Silencer.cpp`](Silencer.cpp.md) · [`WeaponMagazined.cpp`](WeaponMagazined.cpp.md)
**Tier floor** — T3: a declaration only

## Purpose

Declares `CSilencer` as a plain inventory item that overrides the full inventory-item
lifecycle and changes none of it. Substance in [`Silencer.cpp`](Silencer.cpp.md).

Exported units:

- `CSilencer` — the attachment.
- `Load`, `net_Spawn`, `net_Destroy`, `UpdateCL`, `OnH_A_Chield`, `OnH_B_Independent` —
  the lifecycle, all delegating unchanged.
