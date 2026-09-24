# src/xrGame/CarWeapon.cpp

> A turret bolted to a vehicle's skeleton — it slews two bones toward a desired
> direction at a rate-limited speed, refuses to fire until it is on target, and fires
> through the shared shooting machinery.

**Needs** — [`CarWeapon.h`](CarWeapon.h.md) · [`ShootingObject.h`](ShootingObject.h.md) · [`PhysicsShellHolder.h`](PhysicsShellHolder.h.md) · [`WeaponAmmo.h`](WeaponAmmo.h.md) · [`HudSound.h`](HudSound.h.md) · [`game_object_space.h`](game_object_space.h.md) · [`ai_sounds.h`](../xrServerEntities/ai_sounds.h.md) · [`Include/xrRender/Kinematics.h`](../Include/xrRender/Kinematics.h.md) · [`xrPhysics/PhysicsShell.h`](../xrPhysics/PhysicsShell.h.md) · [`xrEngine/xr_level_controller.h`](../xrEngine/xr_level_controller.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: the only sharp edge is that it writes into the pose while the animation
system is composing it, which is a callback ordering constraint, not a memory one.

## Purpose

A vehicle-mounted gun differs from a carried one in exactly two ways, and this file is those
two ways. First, its aim is *the model's own geometry*: two named bones rotate, their travel
limits come from the skeleton's joint limits, and everything else — the muzzle point, the
fire direction, the gunner's camera — is read back out of the resulting pose. Second, it
cannot snap to a target; it slews at a bounded angular speed and is forbidden to fire while
it is still catching up. That combination is what makes a turret feel like a turret.

Everything about ballistics, dispersion, muzzle flash, smoke and light is inherited from the
shared shooting behaviour; this file only supplies the aim, the muzzle frame and the trigger.

## State

```text
RECORD CarWeapon
  owner            : PhysicsShellHolder      # the vehicle
  active           : bool                    # a gunner is controlling it
  auto_fire        : bool                    # fire whenever on target, without a trigger
  allow_fire       : bool                    # recomputed each aim step: are we on target?

  rotate_x_bone    : bone id                 # pitch  (elevation)
  rotate_y_bone    : bone id                 # yaw    (traverse)
  fire_bone        : bone id                 # muzzle frame
  bind_x_rot       : real                    # the bones' rest angles, from the bind pose
  bind_y_rot       : real
  inv_bind_x_xform : matrix                  # inverse bind transforms of the two aim bones
  inv_bind_y_xform : matrix
  lim_x_rot        : (real, real)            # travel limits in bone space, from the skeleton
  lim_y_rot        : (real, real)
  cur_x_rot        : real                    # the angles actually written into the pose
  cur_y_rot        : real
  tgt_x_rot        : real                    # where the angles are heading
  tgt_y_rot        : real

  min_gun_speed    : real                    # slew rate bounds, radians per second
  max_gun_speed    : real
  dest_enemy_dir   : vector                  # the desired aim, in world space
  fire_bone_xform  : matrix                  # muzzle frame in world space, this frame
  fire_pos         : vector                  # muzzle point
  fire_dir         : vector                  # muzzle forward
  fire_norm        : vector                  # muzzle up
  weapon_height    : real                    # yaw bone's height in the bind pose
  ammo             : Cartridge               # one cartridge template, reused every shot
  shot_sound       : sound
```

**Invariants** — `cur_*_rot` are read by the bone callbacks from inside the pose
computation, so they must be settled before the pose is recalculated: the frame update aims
first and recalculates second, never the other way round. `allow_fire` is only meaningful
after an aim step in the same frame.

The three aim bones and the limits are read once at construction. A model whose mount
definition names a bone that does not exist, or whose aim bones carry no joint limits, is a
data error and is not defended against.

## construction

**Contract** — builds the turret from the *vehicle model's* embedded configuration, under
the section `mounted_weapon_definition`. Registers the turret with the per-frame update
scheduler and installs the two bone hooks. Allocates one cartridge template.

```text
FUNCTION construct(vehicle)
  ini = user data of vehicle's visual
  rotate_x_bone, rotate_y_bone, fire_bone = bone ids named in ini
  min_gun_speed, max_gun_speed            = from ini
  lim_x_rot = the pitch bone's joint limits about its own first axis
  lim_y_rot = the yaw   bone's joint limits about its own second axis
  bind = the skeleton's bind-pose transforms
  inv_bind_x_xform = inverse(bind[rotate_x_bone]);  bind_x_rot = pitch of that bind transform
  inv_bind_y_xform = inverse(bind[rotate_y_bone]);  bind_y_rot = yaw   of that bind transform
  cur_x_rot, cur_y_rot = the bind angles            # the turret starts at rest
  dest_enemy_dir = the bind direction, taken into world space
  weapon_height  = height of the yaw bone in the bind pose
  load(ini.mounted_weapon_definition.wpn_section)
  install bone hooks; register for per-frame update
```

**Notes** — the limits come from the *skeleton's* inverse-kinematics data, not from a
configuration file. The modeller who authored the turret already had to constrain those
joints to animate it; reusing that constraint means the gun can never point through its own
mount, and means a mod-added turret is limited correctly with no extra data.

The bind pose is captured inverted because the desired direction arrives in world space and
must be expressed relative to each bone's *rest* frame before it can be turned into an angle
about that bone's axis. Storing the inverses avoids re-inverting every frame.

Construction registers for updates but the destructor deliberately does **not** unregister;
the vehicle's own teardown is what releases the registration, since the vehicle outlives the
turret by construction.

## aiming

**Contract** — the load-bearing algorithm of the file. Once per frame: recompute the muzzle
frame from the current pose, project the desired world direction into each aim bone's rest
frame, clamp to the skeleton's limits, and step the current angles toward the targets at a
speed-limited rate. Sets `allow_fire` as a side effect.

```text
FUNCTION update_barrel_dir()
  fire_bone_xform = pose transform of fire_bone, taken into world space
  fire_pos  = origin  of fire_bone_xform
  fire_dir  = forward of fire_bone_xform
  fire_norm = up      of fire_bone_xform

  d = dest_enemy_dir expressed in the vehicle's own frame

  # pitch: the angle about the pitch bone's axis that points the rest frame at d
  dx = normalize(d in the pitch bone's rest frame)
  tgt_x_rot = normalize_signed(bind_x_rot - pitch_of(dx))
  clamp tgt_x_rot into [-lim_x_rot.high, -lim_x_rot.low]

  # yaw: the same about the yaw bone
  dy = normalize(d in the yaw bone's rest frame)
  tgt_y_rot = normalize_signed(bind_y_rot - yaw_of(dy))
  clamp tgt_y_rot into [-lim_y_rot.high, -lim_y_rot.low]

  cur_x_rot = slew(cur_x_rot, tgt_x_rot, min_gun_speed, max_gun_speed, elapsed)
  cur_y_rot = slew(cur_y_rot, tgt_y_rot, min_gun_speed, max_gun_speed, elapsed)

  allow_fire = cur_x_rot within 5 degrees of tgt_x_rot
           AND cur_y_rot within 5 degrees of tgt_y_rot
```

**Invariants** — the limits are applied *negated and swapped* relative to how the skeleton
stores them, because the bone angle is measured in the opposite sense to the joint's stored
limit. This sign convention is not derivable from the data; it is a property of how the aim
angle is composed into the bone transform (see the bone hooks below) and must be preserved
as a pair with that composition.

**Notes** — `slew` is the engine's shared rate-limited angle interpolation: it moves toward
the target at a speed that scales with the remaining error between a minimum and a maximum
rate, with the scaling saturating at half a turn of error. Fast when far, gentle when close,
and never instantaneous — which is what makes the gun read as heavy.

The five-degree on-target tolerance is generous. It is not an accuracy figure: dispersion is
applied separately at the shot. It is a *gate*, chosen so that a turret tracking a moving
target keeps firing instead of stuttering on and off as the error oscillates.

## the bone hooks

**Contract** — two callbacks the animation system invokes while composing each bone's
transform. Each post-multiplies a rotation about one axis into the bone's transform.

```text
FUNCTION on_pitch_bone(bone)   bone.transform = bone.transform * rotation_about_x(cur_x_rot)
FUNCTION on_yaw_bone(bone)     bone.transform = bone.transform * rotation_about_y(cur_y_rot)
```

**Notes** — this is how the turret aims without an animation: the animation system builds
the rest pose and these hooks bend two joints inside it. Post-multiplication (rotate in the
bone's *own* frame, after its parent chain) is what makes the two axes compose as a
traverse-then-elevate gimbal rather than two world-space rotations that would gimbal-lock
against each other.

The hooks must be uninstalled before the turret goes away; a pose computation that calls a
hook belonging to a destroyed turret is the classic lifetime bug in this pattern.

## the frame update

**Contract** — no-op unless a gunner has activated the turret.

```text
FUNCTION update_frame()
  IF NOT active THEN RETURN
  update_barrel_dir()
  invalidate the pose and recalculate it now      # forced, not deferred
  update_fire()
```

**Notes** — the pose is recalculated *immediately* rather than left to the renderer's lazy
evaluation, because the muzzle frame computed on the next line is read back out of it and is
used this frame to spawn a bullet. A lazily-posed turret would fire from last frame's barrel.

## the firing cycle

**Contract** — advances the shot timer, keeps the muzzle effects alive, and emits a shot
whenever the timer expires while the trigger is held.

```text
FUNCTION update_fire()
  shot_timer = shot_timer - elapsed
  update flame particles and muzzle light
  IF auto_fire
      IF allow_fire THEN fire_start() ELSE fire_end()
  IF NOT firing
      clamp shot_timer at zero            # do not bank up credit while idle
      RETURN
  IF shot_timer <= 0
      on_shot()
      shot_timer = shot_timer + one_shot_time
```

**Invariants** — the timer is *incremented* by the shot interval rather than reset to it, so
a frame longer than the interval still produces the right number of shots over time. The
clamp on the idle path is the matching half: it stops an idle turret accumulating negative
timer and then unloading a burst the instant the trigger is pulled.

**Notes** — auto-fire wires `allow_fire` straight to the trigger. That is the whole of the
turret's "AI": it shoots whenever the barrel has caught up with whatever direction it was
told to aim, and the decision of *where* to aim belongs to whoever is setting the parameter.

## `on_shot`

**Contract** — spawn one bullet from the muzzle frame with the weapon's base dispersion,
then start the muzzle flash, light, smoke and sound. Attributes both the shot and its damage
to the *vehicle*, not to the turret and not to the gunner.

**Notes** — the shot is given a random value in a small range as its per-shot seed. The
bullet system needs a per-shot number to decorrelate dispersion between shots fired in the
same frame; anything that varies will do, which is why the range looks arbitrary.

## `action` and `set_param`

**Contract** — the command surface the vehicle's control code drives.

```text
FUNCTION action(command, on)
  CASE fire:         on ? fire_start() : fire_end()
  CASE activate:     active = on;  IF NOT on THEN fire_end()
  CASE auto_fire:    auto_fire = on
  CASE to_default:   set_param(desired_dir, the bind-pose heading and pitch)

FUNCTION set_param(desired_dir, heading, pitch)   dest_enemy_dir = direction(heading, pitch)
FUNCTION set_param(desired_pos, point)            dest_enemy_dir = normalize(point - fire_pos)
```

**Notes** — deactivating always releases the trigger. Without that, a gunner who dismounts
mid-burst leaves a turret firing forever, since nothing else is left to call `fire_end`.

Aiming at a *point* measures from the muzzle, not from the mount, so the turret converges on
the point rather than on a direction parallel to it — which matters at short range, where the
muzzle offset is a significant fraction of the distance.

## the gunner's camera and on-target query

**Contract** — `view_camera_pos`, `view_camera_dir` and `view_camera_norm` return the muzzle
frame, so the gunner's view is literally down the barrel. `allow_fire` reports the aim gate.
`fire_dir_diff` reports the remaining aim error in degrees, for a user-interface readout.

**Notes** — `fire_dir_diff` compares the *angle pair* treated as a two-dimensional vector
rather than comparing the two directions it implies. That is not the true angular error
between two aim directions, and it degenerates badly when both angles are near zero. It is
only ever used as a coarse display figure, and it is not the value the firing gate uses.
