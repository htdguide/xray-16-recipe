# src/xrGame/WeaponStatMgun.cpp

> The mounted machine gun: a world object the player climbs into rather than carries, whose barrel chases a desired direction through two hinge joints, and whose camera is a bone on its own model.

**Needs** — [`WeaponStatMgun.h`](WeaponStatMgun.h.md) · [`holder_custom.h`](holder_custom.h.md) · [`PhysicsShellHolder.h`](PhysicsShellHolder.h.md) · [`ShootingObject.h`](ShootingObject.h.md) · [`WeaponAmmo.h`](WeaponAmmo.h.md) · [`CameraFirstEye.h`](CameraFirstEye.h.md) · [`Actor.h`](Actor.h.md) · [`Level.h`](Level.h.md) · [`Include/xrRender/Kinematics.h`](../Include/xrRender/Kinematics.h.md) · [Seam: Rigid-body dynamics](../../SYSTEM-REQUIREMENTS.md#seam-rigid-body-dynamics)
**Used by** — reached through its declarations in [`WeaponStatMgun.h`](WeaponStatMgun.h.md); callers name that, not this file.
**Tier floor** — T2: two angle solves and a skeleton pose per frame, plus bone callbacks on the animation path.

## Purpose

Every other weapon in this directory is an inventory item. This one is a **holder**: a
piece of level furniture the actor enters, which then owns his input and his camera until
he leaves. That inversion is the file's reason to exist.

Three consequences follow, and they are the file's substance:

1. **The barrel is posed by the object, not by an animation.** Two bones are hinges — one
   for elevation, one for traverse — and the object writes a rotation into each every
   frame through an animation callback, chasing a desired direction with inertia and
   clamping to the bones' own authored joint limits.
2. **Aiming and firing are decoupled.** The player sets a *desired* direction; the barrel
   chases it. When the barrel has not caught up, or the desired direction is outside the
   mount's limits, firing is suppressed. You cannot shoot where the gun cannot point.
3. **The camera is a bone.** The view is taken from a named bone on the gun's own model
   and eased toward it, so the player's head follows the mount rather than the other way
   around.

## State

```text
RECORD MountedGun EXTENDS PhysicsShellHolder, Holder, ShootingObject
  camera            : Camera             # first-eye, rigidly linked to this object
  ammo              : Cartridge          # one prototype; the gun never runs out

  # --- the mount's skeleton ------------------------------------------
  elevation_bone, traverse_bone, fire_bone, camera_bone : int (16-bit)
  bind_elevation, bind_traverse : real   # each hinge's angle in the bind pose
  inverse_bind_elevation, inverse_bind_traverse : matrix
  elevation_limits, traverse_limits : (real, real)   # from the bones' joint data

  # --- aiming ---------------------------------------------------------
  desired_direction : vector3            # world space; what the player asked for
  target_elevation, target_traverse : real   # desired, resolved into bone space
  current_elevation, current_traverse : real # what the barrel is actually at
  allow_fire        : bool               # the desired direction is within the limits

  # --- firing ---------------------------------------------------------
  fire_position, fire_direction : vector3
  fire_bone_transform : matrix
  spread_offset     : vector2            # accumulated per-shot camera kick, decaying
  camera_max_angle, camera_relax_speed : real

  # --- overheat -------------------------------------------------------
  overheat_enabled  : bool
  overheat_value    : real               # 0 .. threshold
  overheat_rise_per_shot_frame, overheat_fall_per_frame : real
  overheat_threshold : real = 110
  overheat_particles : text
  overheat_effect   : optional<ParticleEmitter>
  firing_disabled   : bool               # latched at the threshold
```

**Invariants** —
- `current_*` chase `target_*` with inertia; they are equal only at rest;
- `allow_fire` is false whenever the desired direction had to be clamped, which is the
  "the mount cannot point there" condition;
- the two bone callbacks are installed only while the actor is mounted and removed when
  he leaves, and installing them **disables the physics shell's own bone callbacks** —
  the two cannot both write a bone's transform.

## `net_Spawn` — reading the mount out of the model

**Contract** — the mount's geometry is not configured; it is **read from the model's own
embedded metadata and joint limits**. That is the interesting decision: an author builds
a new mounted gun entirely in the modelling tool, naming four bones in the model's user
data and setting joint limits on two of them, with no engine-side configuration.

```text
FUNCTION spawn_from_record(gun, record)
  RETURN false IF the base spawn failed
  metadata = the model's embedded user data       # required; a mount without it is an error
  elevation_bone = the bone named by metadata "rotate_x_bone"
  traverse_bone  = the bone named by metadata "rotate_y_bone"
  fire_bone      = the bone named by metadata "fire_bone"
  camera_bone    = the bone named by metadata "camera_bone"

  build a physics shell with the model's root bone fixed        # the mount does not move

  elevation_limits = the elevation bone's joint limits, first axis
  traverse_limits  = the traverse bone's joint limits, SECOND axis

  bind = the model's bind-pose transforms
  inverse_bind_elevation = inverse(bind[elevation_bone])
  inverse_bind_traverse  = inverse(bind[traverse_bone])
  bind_elevation = the pitch of bind[elevation_bone]'s forward axis
  bind_traverse  = the yaw   of bind[traverse_bone]'s forward axis
  current_elevation = bind_elevation; current_traverse = bind_traverse
  desired_direction = the bind pose's forward, rotated into world space

  IF the gun flashes on firing THEN create its light
  register for per-frame updates; make it visible and enabled
```

**Invariants** — the elevation bone's limits are read from its **first** axis and the
traverse bone's from its **second**. Both bones are hinge joints; which axis carries the
limit is a convention of the model format, and getting it wrong gives a gun that cannot
elevate or cannot traverse.

The gun starts pointing wherever its bind pose points, so an author aims a mount by
posing it.

## `UpdateBarrelDir` — the aiming solve

**Contract** — runs every frame. Resolves the world-space desired direction into the two
hinge angles, clamps each to its joint limits, decides whether firing is permitted, and
eases the current angles toward the clamped targets.

```text
FUNCTION update_barrel(gun)
  # 1. where the barrel currently points, in world space
  gun.fire_bone_transform = the fire bone's transform, in world space
  gun.fire_position  = its origin
  gun.fire_direction = its forward axis

  # 2. resolve the desired direction into each hinge's own space
  gun.allow_fire = true
  local = desired_direction, rotated into the gun's own frame

  local = local rotated by inverse_bind_elevation; normalize
  target_elevation = normalize_signed(bind_elevation - pitch(local))
  IF clamping target_elevation to the elevation limits changed it THEN
    gun.allow_fire = false

  local = local rotated by inverse_bind_traverse; normalize
  target_traverse = normalize_signed(bind_traverse - yaw(local))
  IF clamping target_traverse to the traverse limits changed it THEN
    gun.allow_fire = false

  # 3. the barrel chases, it does not snap
  current_elevation = ease(current_elevation, target_elevation,
                           min_speed 0.5, max_speed 3.5, blend_angle 30 degrees)
  current_traverse  = ease(current_traverse,  target_traverse, same)
```

**Invariants** —

- the clamp is applied as `[-limit.max, -limit.min]` — the sign is inverted relative to
  the authored limits because the hinge angles are measured against the bind pose, which
  is also why each target is `bind_angle - measured_angle` rather than the measured angle
  itself;
- the easing is an angular inertia with a floor and a ceiling on speed and a blend zone:
  far from the target it turns at the maximum rate, and within thirty degrees it eases
  down to the minimum. That is what gives a heavy mount its weight — the barrel neither
  snaps nor creeps;
- the elevation solve **mutates the working vector** and the traverse solve then uses the
  mutated one. Reading the second angle from a vector already rotated into the first
  bone's space is either a subtle correctness argument about the hinge chain or a bug; it
  is not recoverable from the source, and the behaviour it produces is what shipped.

## The bone callbacks

**Contract** — the two hinge angles reach the skeleton through callbacks attached to the
two bones, which post-multiply a rotation onto each bone's computed transform every time
the pose is built.

```text
ON elevation bone posed:  bone.transform = bone.transform * rotation_about_X(current_elevation)
ON traverse bone posed:   bone.transform = bone.transform * rotation_about_Y(current_traverse)
```

**Invariants** — installing them turns the physics shell's own callbacks **off**, and
removing them turns the shell's back on. Both want to write the same bones; only one may.
The callbacks are installed when the actor mounts and removed when he dismounts, so an
unmanned gun is posed by physics and a manned one by the aiming solve.

**Notes** — this is a *post*-multiplication, so the hinge rotation is applied in the
bone's local frame on top of whatever the animation or physics produced. A rebuild that
drives the pose directly must reproduce that composition order.

## `cam_Update` — the camera is a bone

**Contract** — the view is taken from the camera bone's world transform and eased toward
it, and the actor's head is then slaved to the resulting camera orientation.

```text
FUNCTION update_camera(gun, frame_time, field_of_view)
  rebuild the gun's pose                         # forced: the camera bone must be current
  bone = the camera bone's transform
  position  = bone origin, in world space
  direction = bone forward, in world space
  desired_yaw, desired_pitch = the heading and pitch of direction, both NEGATED

  camera.yaw   = ease(camera.yaw,   desired_yaw,   0.5, 7.5, 30 degrees, frame_time)
  camera.pitch = ease(camera.pitch, desired_pitch, 0.5, 7.5, 30 degrees, frame_time)

  IF an actor is mounted THEN
    actor.orientation.yaw   = -camera.yaw       # the head follows the camera
    actor.orientation.pitch = -camera.pitch

  update the camera at position, and hand it to the level's camera stack
```

**Invariants** — the camera eases at more than twice the barrel's maximum rate (7.5
against 3.5), so the view leads the barrel. The player sees where the gun is going before
it gets there, which is what makes a slow mount usable.

The double negation — the bone's angles are negated into the camera, and the camera's are
negated back into the actor — is a sign convention between the camera's and the model's
frames, not a transformation.

The pose is force-rebuilt at the top because the camera must read *this* frame's bone,
after the aiming solve wrote the hinge angles.

## Input

**Contract** — while mounted, the gun receives input directly. Every path funnels into
one procedure that moves the *desired direction*, never the barrel.

```text
FUNCTION on_axis_move(gun, dx, dy, scale_x, scale_y, invert_x, invert_y)
  RETURN IF both are zero
  yaw, pitch = the heading and pitch of gun.desired_direction
  yaw   = yaw   - (invert_x ? -1 : 1) * dx * scale_x
  pitch = pitch - (invert_y ? -1 : 1) * dy * scale_y * 3/4
  gun.desired_direction = direction from (yaw, pitch)
```

**Invariants** — vertical input is scaled by three quarters relative to horizontal. That
is a deliberate feel choice for a heavy mount, not an aspect correction, and it applies to
mouse, stick and motion input alike.

Mouse, gamepad stick and controller motion each supply their own sensitivity scale and
inversion flags from the player's settings and then call the same procedure. The fire
binding starts and stops firing on press and release. A remote player's gun ignores all
input; its direction arrives over the network.

## `Action` / `SetParam`

**Contract** — the holder interface's two entry points. `Action` handles the fire
binding, so that a script or the AI can fire the gun without synthesizing input.
`SetParam` accepts a desired direction as a heading/pitch pair under a numbered
parameter, which is how a stalker aims a mounted gun.

## Mounting

**Contract** — attaching the actor installs the bone callbacks and ends any firing;
detaching removes them and ends firing again. The gun declares it may be used only when
nobody is in it, that a mounted actor may not use his own weapons, that the view is
first-person, and that it provides no inventory.

Two authored flags can lock entry or exit — a scripted turret the player must use, or
must not.

**Invariants** — `Hit` is ignored entirely while somebody is mounted: a manned gun takes
no damage. That protects the occupant and is a deliberate gameplay decision, not an
oversight.

## Replication

**Contract** — the payload adds the firing flag and the desired direction to the physics
holder's. On import, the firing flag's *transitions* are turned back into start and stop
calls, so the remote gun's particles, sound and recoil run through the same paths as a
local one's. The desired direction is replicated rather than the barrel angles, so each
machine runs its own easing and the barrel's motion stays smooth under packet loss.

## `UpdateCL`

**Contract** — the frame order is: base physics update, barrel solve, firing update, and
then — only when the mounted actor's camera is the active one — the camera update, pushed
into the actor's camera stack and applied to the device. The camera must run last because
it reads the bones the barrel solve just moved.
