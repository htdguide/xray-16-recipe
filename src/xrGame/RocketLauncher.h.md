# src/xrGame/RocketLauncher.h

> Declares the entity-projectile launcher mixin implemented in [`RocketLauncher.cpp`](RocketLauncher.cpp.md).

**Needs** — [`CustomRocket.h`](CustomRocket.h.md)
**Used by** — [`Helicopter.cpp`](Helicopter.cpp.md) · [`HelicopterWeapon.cpp`](HelicopterWeapon.cpp.md) · [`RocketLauncher.cpp`](RocketLauncher.cpp.md) · [`WeaponMagazinedWGrenade.cpp`](WeaponMagazinedWGrenade.cpp.md) · [`WeaponMagazinedWGrenade.h`](WeaponMagazinedWGrenade.h.md) · [`WeaponRG6.cpp`](WeaponRG6.cpp.md) · [`WeaponRG6.h`](WeaponRG6.h.md) · [`WeaponRPG7.cpp`](WeaponRPG7.cpp.md) · [`WeaponRPG7.h`](WeaponRPG7.h.md) · [`helicopter.h`](helicopter.h.md)
**Tier floor** — T3: a declaration only

## Purpose

Declares `CRocketLauncher`, the mixin a weapon or turret inherits alongside its own base to
gain the ability to carry and release spawned projectile entities. Substance is in
[`RocketLauncher.cpp`](RocketLauncher.cpp.md).

It holds two lists — projectiles loaded and projectiles in flight — and the configured
muzzle velocity.

Exported units:

- `CRocketLauncher` — the mixin.
- `Load` — reads the muzzle velocity from the section.
- `SpawnRocket` — asks the authority to create a projectile of a named section, parented
  to this launcher.
- `AttachRocket` — adopts an arrived projectile by identifier.
- `DetachRocket` — releases a projectile by identifier, flagging whether it was launched.
- `LaunchRocket` — gives the current projectile its starting transform and velocities.
- `getCurrentRocket`, `dropCurrentRocket`, `getRocketCount` — the loaded stack's top, pop
  and size.
