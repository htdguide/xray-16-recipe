# src/xrGame/aimers_weapon_inline.h

> The weapon aimer's single accessor.

**Needs** — [`aimers_weapon.h`](aimers_weapon.h.md)
**Used by** — [`aimers_weapon.cpp`](aimers_weapon.cpp.md) · [`aimers_weapon.h`](aimers_weapon.h.md)
**Tier floor** — T3: array access.

## Purpose

Returns one of the two carrier-bone corrections by index, asserting the index is in range.
The file exists only because the original language wants the definition after the class
body; a rebuild has nothing to put here.
