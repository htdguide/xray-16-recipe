# src/xrGame/ShootingObject.h

> Declares the firing mixin implemented in [`ShootingObject.cpp`](ShootingObject.cpp.md), and the small record of silencer multipliers.

**Needs** — [`alife_space.h`](../xrServerEntities/alife_space.h.md) · [`anticheat_dumpable_object.h`](anticheat_dumpable_object.h.md) · [`xrEngine/Render.h`](../xrEngine/Render.h.md)
**Used by** — [`CarWeapon.cpp`](CarWeapon.cpp.md) · [`CarWeapon.h`](CarWeapon.h.md) · [`Helicopter.cpp`](Helicopter.cpp.md) · [`HelicopterWeapon.cpp`](HelicopterWeapon.cpp.md) · [`ShootingObject.cpp`](ShootingObject.cpp.md) · [`Weapon.cpp`](Weapon.cpp.md) · [`Weapon.h`](Weapon.h.md) · [`WeaponStatMgun.cpp`](WeaponStatMgun.cpp.md) · [`WeaponStatMgun.h`](WeaponStatMgun.h.md) · [`WeaponStatMgunFire.cpp`](WeaponStatMgunFire.cpp.md) · [`helicopter.h`](helicopter.h.md) · [`shootingObject_dump_impl.cpp`](shootingObject_dump_impl.cpp.md) · [`weapon_dump_impl.cpp`](weapon_dump_impl.cpp.md)
**Tier floor** — T3: a declaration only

## Purpose

Declares `CShootingObject`, the mixin every weapon, vehicle gun and fragmenting grenade
inherits to gain rate of fire, damage, dispersion, muzzle light and muzzle effects.
Substance is in [`ShootingObject.cpp`](ShootingObject.cpp.md).

It also declares the one record it owns outright:

- **the silencer multiplier set** — six factors applied to hit power, hit impulse, bullet
  speed, fire dispersion, camera dispersion and camera dispersion growth. All six reset to
  one, which is the neutral element, so the firing path multiplies unconditionally. Two
  instances are held: what a fitted silencer would supply, and what is currently in force.

Three demands it makes of whatever it is mixed into, which are the mixin's real interface:
`IsHudModeNow` (is this object drawn as a first-person model right now),
`get_CurrentFirePoint` (where the muzzle is in world space) and `get_ParticlesXFORM` (the
orientation effects inherit). A fourth, `ForceUpdateFireParticles`, is optional.

Exported units:

- `CShootingObject` — the mixin.
- `Load`, `LoadFireParams`, `LoadLights`, `LoadShellParticles`, `LoadFlameParticles`,
  `reinit`, `reload` — configuration and re-spawn.
- `FireBullet`, `FireStart`, `FireEnd`, `IsWorking` — the shot itself.
- `SendHitAllowed` — which machine computes a shot's damage.
- `SetBulletSpeed`, `GetBulletSpeed` — post-load muzzle velocity override.
- `ParentMayHaveAimBullet`, `ParentIsActor` — two questions about the shooter, defaulting
  to no.
- `Light_Create`, `Light_Destroy`, `Light_Start`, `Light_Render`, `UpdateLight`,
  `StopLight`, `RenderLight` — the muzzle flash.
- `StartParticles`, `UpdateParticles`, `StopParticles` — the generic effect slot.
- `StartFlameParticles`, `UpdateFlameParticles`, `StopFlameParticles` — the one held
  effect.
- `StartSmokeParticles`, `StartShotParticles`, `OnShellDrop` — the fire-and-forget
  effects.
- `DumpActiveParams` — writes the object's live parameters back in configuration form for
  the anti-cheat comparison; implemented in a separate module.

## Notes

The header names the surface material used for bullet impacts as a compile-time constant
path into the material table. It is data, frozen by the shipped material file, and belongs
with the ballistics manager that reads it.

`reload` is declared with an empty body and no caller — a hook from an older design where
a weapon could be reconfigured mid-life. A rebuild should not carry it forward.
