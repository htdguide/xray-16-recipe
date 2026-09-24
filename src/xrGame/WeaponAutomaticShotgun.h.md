# src/xrGame/WeaponAutomaticShotgun.h

> Declares the automatic shotgun implemented in [`WeaponAutomaticShotgun.cpp`](WeaponAutomaticShotgun.cpp.md).

**Needs** — [`WeaponMagazined.h`](WeaponMagazined.h.md) · [`WeaponShotgun.h`](WeaponShotgun.h.md)
**Used by** — [`WeaponAutomaticShotgun.cpp`](WeaponAutomaticShotgun.cpp.md)
**Tier floor** — T3: a declaration only

## Purpose

Declares `CWeaponAutomaticShotgun`: full-automatic fire from the magazined weapon plus a
copy of the shell-at-a-time reload sub-state machine. Substance is in
[`WeaponAutomaticShotgun.cpp`](WeaponAutomaticShotgun.cpp.md), which defers to
[`WeaponShotgun.cpp`](WeaponShotgun.cpp.md) for the sequence itself.

The declared surface is the shotgun's reload surface, member for member: `Load`,
`Reload`, `TriStateReload`, `OnStateSwitch`, `OnAnimationEnd`, the three `switch2_*`
phases, the three `PlayAnim*` procedures, `HaveCartridgeInInventory`, `AddCartridge`,
`Action`, and the per-shell `net_Export`/`net_Import`. Only `switch2_Fire` is absent,
because firing comes from the automatic base.

**Notes** — the header includes the shotgun's header without deriving from it, purely to
reach its declarations. That include is the residue of the copy.
