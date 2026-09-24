# src/xrGame/WeaponStatMgun.h

> Declares the mounted machine gun, implemented across [`WeaponStatMgun.cpp`](WeaponStatMgun.cpp.md), [`WeaponStatMgunFire.cpp`](WeaponStatMgunFire.cpp.md) and [`WeaponStatMgunIR.cpp`](WeaponStatMgunIR.cpp.md).

**Needs** — [`holder_custom.h`](holder_custom.h.md) · [`ShootingObject.h`](ShootingObject.h.md) · [`PhysicsShellHolder.h`](PhysicsShellHolder.h.md) · [`HudSound.h`](HudSound.h.md)
**Used by** — [`WeaponStatMgun.cpp`](WeaponStatMgun.cpp.md) · [`WeaponStatMgunFire.cpp`](WeaponStatMgunFire.cpp.md) · [`WeaponStatMgunIR.cpp`](WeaponStatMgunIR.cpp.md)
**Tier floor** — T3: a declaration only

## Purpose

Declares `CWeaponStatMgun`. It is deliberately **not** a `CWeapon`: it is a physics
object, a *holder* (something the actor climbs into) and a shooting object, with no
inventory item anywhere in its ancestry. Substance is split across three files — the
mount and camera in [`WeaponStatMgun.cpp`](WeaponStatMgun.cpp.md), the shot and overheat
in [`WeaponStatMgunFire.cpp`](WeaponStatMgunFire.cpp.md), the input in
[`WeaponStatMgunIR.cpp`](WeaponStatMgunIR.cpp.md).

One named constant is part of the holder protocol: parameter 1 is the desired direction,
set as a heading/pitch pair.

Exported units, by group:

**Lifecycle** — `Load` (camera recoil, overheat and entry-lock parameters), `net_Spawn`
(read the four bones and two joint limits out of the model), `net_Destroy`, `net_Export`
/ `net_Import` (firing flag plus desired direction), `UpdateCL`, `Hit` (ignored while
manned), `renderable_Render`.

**Aiming** — `UpdateBarrelDir`, `SetDesiredDir`, `SetBoneCallbacks` /
`ResetBoneCallbacks` and the two static bone callbacks, `cam_Update`, `Camera`.

**Firing** — `FireStart`, `FireEnd`, `UpdateFire` (the shot clock and the overheat
model), `OnShot`, `AddShotEffector` / `RemoveShotEffector`, `get_CurrentFirePoint`,
`get_ParticlesXFORM`.

**Holder interface** — `Use` (only when unoccupied), `attach_Actor` / `detach_Actor`,
`allowWeapon` (false: a mounted actor may not use his own), `HUDView` (first person),
`GetInventory` (none), `ExitPosition`, `Action`, `SetParam`, `cast_holder_custom`.

**Input** — `OnAxisMove` (the common path; vertical scaled to three quarters),
`OnMouseMove`, the three keyboard hooks, the three controller hooks, and
`OnControllerAttitudeChange` for motion input.
