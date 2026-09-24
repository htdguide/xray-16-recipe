# src/xrGame/WeaponShotgun.h

> Declares the manually cycled shotgun implemented in [`WeaponShotgun.cpp`](WeaponShotgun.cpp.md).

**Needs** — [`WeaponCustomPistol.h`](WeaponCustomPistol.h.md)
**Used by** — [`WeaponAutomaticShotgun.cpp`](WeaponAutomaticShotgun.cpp.md) · [`WeaponAutomaticShotgun.h`](WeaponAutomaticShotgun.h.md) · [`WeaponRG6.cpp`](WeaponRG6.cpp.md) · [`WeaponRG6.h`](WeaponRG6.h.md) · [`WeaponScript.cpp`](WeaponScript.cpp.md) · [`WeaponShotgun.cpp`](WeaponShotgun.cpp.md) · [`death_anims_predicates.cpp`](death_anims_predicates.cpp.md) · [`weaponBM16.h`](weaponBM16.h.md)
**Tier floor** — T3: a declaration only

## Purpose

Declares `CWeaponShotgun`: a semi-automatic weapon whose reload is a three-phase
shell-at-a-time sequence and whose magazine contents are replicated. Substance is in
[`WeaponShotgun.cpp`](WeaponShotgun.cpp.md).

Exported units:

- `Load` — reads the tri-state-reload flag and, if set, three extra sounds.
- `Reload` / `TriStateReload` — enter the shell sequence, or fall back to the ordinary
  reload when the weapon is not authored as tri-state.
- `OnStateSwitch` / `OnAnimationEnd` — dispatch and advance the sub-state machine.
- `switch2_StartReload` / `switch2_AddCartgidge` / `switch2_EndReload` — the three
  phases.
- `PlayAnimOpenWeapon` / `PlayAnimAddOneCartridgeWeapon` / `PlayAnimCloseWeapon`.
- `HaveCartridgeInInventory(n)` — availability, with the side effect of switching ammo
  type when the current one runs out.
- `AddCartridge(n)` — push shells; returns how many could not be pushed.
- `Action` — fire during the shell phase aborts the reload after one more shell.
- `switch2_Fire` — the semi-automatic arming.
- `net_Export` / `net_Import` — replicate the magazine's per-shell ammo types.
