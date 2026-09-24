# src/xrGame/CarWeapon.h

> Declares the turret weapon a vehicle carries — the surface implemented in
> [`CarWeapon.cpp`](CarWeapon.cpp.md).

**Needs** — [`ShootingObject.h`](ShootingObject.h.md) · [`HudSound.h`](HudSound.h.md)
**Used by** — [`Car.cpp`](Car.cpp.md) · [`CarInput.cpp`](CarInput.cpp.md) · [`CarScript.cpp`](CarScript.cpp.md) · [`CarWeapon.cpp`](CarWeapon.cpp.md)
**Tier floor** — T2: a declaration page.

## Purpose

Declares `CarWeapon`, the mounted gun bolted to a vehicle's skeleton. It is a *shooting
object* — it inherits the firing, dispersion, particle and muzzle-light machinery — but not
an inventory item and not a game object; the vehicle owns it directly.

## `CarWeapon`

The exported surface, implemented in [`CarWeapon.cpp`](CarWeapon.cpp.md):

- **construct from the owning physics-shell holder** — reads the mount definition out of the
  vehicle model's embedded configuration and installs the aim bone hooks.
- **`load(section)`** — pull the weapon's ballistic and audio tuning from a configuration
  section.
- **`update_frame`** — aim, force a pose recalculation, then advance the firing cycle.
- **`action(id, on_off)`** — the command surface: activate, fire, auto-fire, return to the
  rest direction. The identifiers are an enumeration declared here, so they are part of the
  contract between the vehicle's control code and the turret.
- **`set_param(id, …)`** — set the desired aim, either as a heading/pitch pair or as a world
  point to aim at.
- **`allow_fire`** / **`fire_dir_diff`** — is the barrel on target, and by how many degrees.
- **`view_camera_pos` / `_dir` / `_norm`** — the turret's own camera frame, so a gunner can
  look down the barrel.
- **`height`** — the mount's height above the vehicle origin, used to place that camera.
- **`is_active`** — whether the turret is currently under a gunner's control.
- **`render_internal`** — draws the muzzle light.
- **bone hooks (static)** — two callbacks the animation system invokes while composing the
  pose, one per aim axis.

**Notes** — the turret declares that it is never in "heads-up display mode". A hand-held
weapon uses that flag to decide whether it is drawn in a first-person rig with its own
animation set; a turret is always part of the world model, never a first-person prop.
