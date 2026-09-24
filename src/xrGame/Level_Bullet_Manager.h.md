# src/xrGame/Level_Bullet_Manager.h

> Declares the bullet record and the central projectile simulation, implemented in [`Level_Bullet_Manager.cpp`](Level_Bullet_Manager.cpp.md) and [`Level_bullet_manager_firetrace.cpp`](Level_bullet_manager_firetrace.cpp.md).

**Needs** — [`WeaponAmmo.h`](WeaponAmmo.h.md) · [`Tracer.h`](Tracer.h.md) · [`xrCDB/xr_collide_defs.h`](../xrCDB/xr_collide_defs.h.md) · [`xrSound/Sound.h`](../xrSound/Sound.h.md)
**Used by** — [`Explosive.cpp`](Explosive.cpp.md) · [`Level.cpp`](Level.cpp.md) · [`Level_Bullet_Manager.cpp`](Level_Bullet_Manager.cpp.md) · [`Level_bullet_manager_firetrace.cpp`](Level_bullet_manager_firetrace.cpp.md) · [`Level_network.cpp`](Level_network.cpp.md) · [`Level_start.cpp`](Level_start.cpp.md) · [`ShootingObject.cpp`](ShootingObject.cpp.md) · [`WeaponAmmo.cpp`](WeaponAmmo.cpp.md) · [`WeaponFire.cpp`](WeaponFire.cpp.md) · [`WeaponKnife.cpp`](WeaponKnife.cpp.md) · [`game_cl_base_weapon_usage_statistic.cpp`](game_cl_base_weapon_usage_statistic.cpp.md) · [`game_cl_base_weapon_usage_statistic.h`](game_cl_base_weapon_usage_statistic.h.md)
**Tier floor** — T3: a declaration

## Purpose

Declares `SBullet` — one projectile in flight — and `CBulletManager`, which owns all of them.
Substance is split: the trajectory model, the sweep and the tracer drawing are in
[`Level_Bullet_Manager.cpp`](Level_Bullet_Manager.cpp.md); what a hit *does* to a material,
an armour value or a bone is in
[`Level_bullet_manager_firetrace.cpp`](Level_bullet_manager_firetrace.cpp.md).

Two shape decisions here outlive any implementation.

**A bullet is a plain record in a flat array, not an object.** It has no identity in the
object registry, no scheduler slot, no network presence and no destructor. Everything a
projectile needs is 150-odd bytes, and the manager owns the array. A rebuild that makes
bullets entities will not reach the same projectile counts.

**Hits are deferred, not applied.** Discovering a hit and acting on it are separated by a
frame boundary: the sweep may run on a worker thread and may not touch the world, so it
records an event and the main thread applies it at the top of the next frame. The event
carries a *copy* of the bullet, because the original will have moved on.

Exported units:

- `SBullet_Hit` — the pair a hit transfers: damage power and physical impulse. They are
  separate because armour reduces one and not the other.
- `SBullet` — the projectile: provenance (firer, weapon, target, frame of birth), kinematics
  (segment start position and velocity, elapsed segment time, current position, direction and
  speed, total distance flown, segment count), cartridge-derived constants (range, drag,
  armour piercing, material piercing, wallmark size, tracer colour, bullet material), hit
  type, and seven behaviour flags — ricocheted already, explosive, may leave a tracer, may
  ricochet, report hits to statistics, was an aimed first shot, and travels as a magnetic
  beam that neither deflects nor slows on penetration.
- `SBullet::CanBeRenderedNow` — false on the frame the bullet was created, so a tracer is
  never a zero-length streak at the muzzle.
- `CBulletManager` — the simulation.
- `Load` / `Clear` — read the tuning set (from a separate configuration section in
  multiplayer), and drop everything on a level change.
- `AddBullet` — create one from a shot and a cartridge. Main thread only.
- `CommitEvents` — at the **start** of a frame: apply every deferred hit and removal.
- `CommitRenderSet` — at the **end** of a frame: copy the working set for drawing, then run
  or schedule the sweep.
- `Render` — draw the tracers as one batch.
- `UpdateWorkload` — the sweep over every bullet; runs on a worker thread when configured.
- `process_bullet` / `trajectory_check_error` / `add_bullet_point` — one bullet's advance:
  adaptive segmentation of the curve, one ray query per segment, and a debug polyline.
- `firetrace_callback` / `test_callback` — the ray query's result handler and its candidate
  filter. Static so they can be handed to the collision database.
- `RegisterEvent` — record a deferred hit or removal, copying the bullet.
- `ObjectHit` / `DynamicObjectHit` / `StaticObjectHit` / `FireShotmark` — hit resolution and
  decal placement.
- `PlayWhineSound` / `PlayExplodePS` — the two presentation side effects of an impact.
- `bullet_test_callback_data` — the scratch record threaded through a ray query: the impact
  position, the bullet, the time of impact along the curve, and the segment's end time.

## Notes

The tuning constants are protected fields rather than a named record: gravity, a global drag
coefficient, a minimum speed, two collision-energy bounds, a hit-probability range, and three
tracer dimensions. Naming them as a settings record would make the multiplayer override —
which reads the same names from a different configuration section — obvious rather than
implicit.

`parent_ignore_distance` (3 units) is the distance within which a bullet cannot hit its own
firer; below it the firer is filtered out of the ray query entirely. It is what stops a
muzzle-adjacent shot killing the shooter.

The event's `tgt_material` field carries the **array index** on a removal event rather than a
material. That overload is why the sweep must iterate the array in reverse.
