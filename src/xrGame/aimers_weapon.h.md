# src/xrGame/aimers_weapon.h

> Declares the weapon aimer: two carrier bones, two weapon anchor bones and their parent, and the pair of corrections it produces.

**Needs** — [`aimers_weapon.cpp`](aimers_weapon.cpp.md) · [`aimers_base.h`](aimers_base.h.md) · [`aimers_weapon_inline.h`](aimers_weapon_inline.h.md)
**Used by** — [`aimers_weapon.cpp`](aimers_weapon.cpp.md) · [`aimers_weapon_inline.h`](aimers_weapon_inline.h.md) · [`sight_manager.cpp`](sight_manager.cpp.md)
**Tier floor** — T3: a declaration.

## Purpose

Declares the surface implemented in [`aimers_weapon.cpp`](aimers_weapon.cpp.md).

Exported units:

- **construction** from the carrier object, the candidate motion, a first-or-last-frame
  flag, the target, the two carrier bone names, the two weapon anchor bone names, and the
  weapon itself;
- **`get_bone`** — one of the two corrections, indexed 0 or 1.

**Notes** — The five bone slots are named by an enumeration whose order matters: the two
carrier links come first so that a loop over "the bones to be corrected" is a loop over the
prefix, and the assertion in the solver is simply that the link index is below the first
weapon anchor's slot. A rebuild should keep the grouping or make the two sets separate
arrays.
