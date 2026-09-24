# src/xrGame/WeaponRPG7.h

> Declares the rocket launcher implemented in [`WeaponRPG7.cpp`](WeaponRPG7.cpp.md).

**Needs** — [`WeaponCustomPistol.h`](WeaponCustomPistol.h.md) · [`RocketLauncher.h`](RocketLauncher.h.md)
**Used by** — [`WeaponRPG7.cpp`](WeaponRPG7.cpp.md)
**Tier floor** — T3: a declaration only

## Purpose

Declares `CWeaponRPG7`, a single-shot weapon that is simultaneously a semi-automatic
firearm and a projectile launcher. Substance is in
[`WeaponRPG7.cpp`](WeaponRPG7.cpp.md).

The declaration's own content is the combination: firing behaviour from the
semi-automatic base, projectile spawning and launching from the launcher base. A rebuild
should make the launcher a member rather than a base.

Exported units:

- `Load` — both halves, plus the launcher's own zoom-factor key and rocket class name.
- `net_Spawn` — restore the rocket object behind a loaded round.
- `switch2_Fire` — the launch, with the muzzle-parallax correction.
- `FireTrace` — the ordinary shot, then refresh rocket visibility.
- `UpdateMissileVisibility` — the bone visibility rule, called from nine places.
- `ReloadMagazine` / `UnloadMagazine` — keep the rocket object in step.
- `OnStateSwitch` / `net_Import` / `on_a_hud_attach` — refresh visibility.
- `OnEvent` — rocket ownership taken, rejected or launched.
- `AllowBore` — suppress the idle fidget when empty.
- `PlayAnimReload` — reload without blending into the current pose.
- `SwitchState`, `FireStart` — empty overrides.

Registered to the script virtual machine as a subclass of the magazined weapon.
