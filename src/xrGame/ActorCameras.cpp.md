# src/xrGame/ActorCameras.cpp

> Places the player's eye each frame: on the body at the right height, leaning around corners only as far as the wall allows, smoothed over stairs, and pushed out of geometry it would otherwise see through.

**Needs** — [`Actor.h`](Actor.h.md) · [`ActorEffector.h`](ActorEffector.h.md) · [`SleepEffector.h`](SleepEffector.h.md) · [`EffectorShot.h`](EffectorShot.h.md) · [`CharacterPhysicsSupport.h`](CharacterPhysicsSupport.h.md) · [`PHMovementControl.h`](PHMovementControl.h.md) · [`IKLimbsController.h`](IKLimbsController.h.md) · [`Level.h`](Level.h.md) · [`GamePersistent.h`](GamePersistent.h.md) · [`Car.h`](Car.h.md) · [`xrEngine/CameraBase.h`](../xrEngine/CameraBase.h.md) · [`xrCDB/Intersect.hpp`](../xrCDB/Intersect.hpp.md) · [`xrPhysics/IElevatorState.h`](../xrPhysics/IElevatorState.h.md) · [`xrPhysics/ActorCameraCollision.h`](../xrPhysics/ActorCameraCollision.h.md) · [Seam: Static collision database](../../SYSTEM-REQUIREMENTS.md#seam-static-collision-database)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: per-frame transform work plus swept box queries against the collision database

## Purpose

The camera is not simply "the actor's position plus eye height". Five corrections are
applied on top of that, and each exists because without it the player sees something
wrong: the **lean** must stop at a wall rather than putting the eye inside it; **stairs**
must not be climbed as a series of vertical jumps; a **ladder** must lock the view roughly
along the ladder; **foot placement** must not translate into a bobbing eye; and the eye's
near plane must never end up **inside geometry**, because that shows the player the
inside of the world.

The actor owns several cameras — first person, third person behind, third person in front,
free look — and switches between them, but they all read the same computed eye point. Only
the first-person camera is corrected for collision, because only it is ever the one the
device renders from.

## State

The state lives on the actor; this file is the rules that move it.

```text
RECORD ActorCameraState
  cam_active            : enum {first_eye, look_at, free_look, fixed}
  cameras[4]            : camera            # each owns its own yaw/pitch/roll and their limits
  current_height        : real              # smoothed eye height; < 0 means "not yet initialised"
  previous_cam_y        : real              # the stair-smoothing reference height
  current_ik_shift      : real              # smoothed foot-placement vertical offset
  previous_cam_dir      : vector            # last frame's view direction
  angular_velocity      : real              # derived: how fast the view is turning, in radians per second
  torso_roll            : real              # the lean actually granted
  torso_target_roll     : real              # the lean the player asked for
```

**Invariants** — the granted lean never exceeds the requested lean and always has the same
sign. The smoothed eye height is initialised on its first use to the target rather than to
zero, so the first frame of a level does not interpolate up from the actor's feet.

### Tuning constants

```text
ladder_yaw_limit          = 1.0 rad     # half-width of the view cone while on a ladder
camera_into_collision     = 0.05 m      # the eye sits this far below the collision box's top
stair_smoothing_rate      = 1.5 m/s     # how fast the eye catches up after a step up
stair_smoothing_max_lag   = 0.2 m       # the eye never trails the body by more than this
height_interpolation_rate = 4.0 /s      # crouch and stand transition rate
ik_shift_tolerance        = 0.2 m
lean_search_step          = PI/1000     # angular resolution of the lean sweep
```

## `cam_Set`

**Contract** — switches the active camera. The outgoing camera is deactivated first and
the incoming one is activated *with the outgoing one as its argument*, so it can inherit
its yaw and pitch. Without that handoff, changing view mode snaps the view.

## `cam_Update`

**Contract** — the per-frame entry point. Does nothing at all when the actor is in a
holder: a vehicle or turret owns the camera while occupied. Otherwise it computes the eye
point and hands it to the active camera, and — only when this actor is the one the device
renders from — applies the result to the device.

```text
FUNCTION cam_update(dt, field_of_view)
  IF in a holder THEN RETURN                      # the holder drives the camera

  IF climbing AND not in free look THEN update_ladder_constraint(dt)
  advance the weapon-recoil effector

  # Foot-placement compensation, multiplayer only — see Notes
  ik_shift = foot placement controller's current vertical offset, smoothed
  smooth current_height toward camera_height()    # crouch/stand transition

  eye = (0, current_height + ik_shift, 0) in a frame yawed to the torso and
        translated to the actor's position

  IF this actor has control THEN resolve_lean(frame, eye.y)
  IF leaning THEN
    displace the eye along the lean arc and derive a roll angle from it

  smooth_stair_step(eye)                          # see below
  transform the eye point into world space

  active_camera.update(eye, angles); active_camera.fov = field_of_view
  IF the active camera is not the first-person one THEN
    update the first-person camera to the same point anyway   # see Notes

  IF this actor is the rendered entity THEN
    push the first-person camera out of any geometry its near plane intersects
  fold in the effector stack

  angular_velocity = |view direction - previous view direction| / frame_seconds

  IF this actor is the rendered entity
     AND the active camera is first-person
     AND no cinematic or photo-mode effector holds the camera THEN
    write the result to the device
```

**Invariants** — the device is written at most once per frame and only by the rendered
entity's camera. A cinematic or photo-mode effector takes the camera away entirely by
being present; this is the single-owner rule the engine's exclusive-signal mechanism
expresses elsewhere, applied here by an explicit check.

**Notes**

- The **first-person camera is updated even when it is not active** because it is the
  source of the view direction used for angular velocity, for weapon aiming and for the
  collision push-out. A rebuild can make that explicit by separating "the aiming eye" from
  "the rendered camera" — they are the same thing in this engine only by convention.
- The **foot-placement compensation is gated to multiplayer**, which is an odd gate: the
  same inverse-kinematics controller runs in single player, and its vertical shift is
  simply ignored there. The reason is not recoverable from the source; the effect in
  single player is that the eye does not dip when a foot lands on a lower step.
- Its smoothing is a **two-rate filter**: a fast rate while the shift is far from where the
  camera currently sits, and a much slower one once it is small, so the eye settles without
  a visible creep. Both rates are expressed as a per-frame blend factor derived from the
  frame time and a constant, which makes them frame-rate independent.

### stair smoothing

**Contract** — while the actor is standing on ground *and* has risen since the last frame,
the eye is allowed to rise at a fixed rate rather than instantly, and never trails the body
by more than a fixed distance. In the air, or when falling, the eye tracks the body exactly.

```text
IF on ground AND body_y > previous_cam_y THEN
  previous_cam_y += stair_smoothing_rate * dt
  clamp previous_cam_y to at most body_y
  clamp previous_cam_y to at least body_y - stair_smoothing_max_lag
  eye.y += previous_cam_y - body_y                 # a negative offset: the eye lags
ELSE
  previous_cam_y = body_y                          # no smoothing downward, ever
```

**Notes** — smoothing only upward is the decision. Falling is supposed to feel abrupt; a
step up is supposed to feel like a step, not a lurch. The maximum lag is what bounds the
error when the actor runs up a long staircase faster than the smoothing rate.

## `cam_Lookout`

**Contract** — grants a lean. Given the requested lean angle, sweeps from zero outward in
small angular increments, testing at each step whether the camera's *near-plane box*,
oriented as it would be at that lean, intersects any impassable geometry. The first
blocked angle is the limit; if nothing is blocked, the full request is granted. Called
only for the actor that currently has control.

**Invariants** — the actor's roll is set to twice the granted angle, because the lean is
modelled as the eye travelling along an arc of radius half the eye height: the eye's
rotation about the body is half the body's roll.

```text
FUNCTION resolve_lean(body_frame, eye_height)
  IF requested lean is zero THEN clear both roll fields; RETURN

  half_angle = requested_roll / 2
  radius     = eye_height / 2
  near_box   = half-extents of the camera's near plane, plus its depth

  # Fast path: if the fully leant position is already clear, grant it outright.
  IF NOT blocked_at(half_angle) THEN grant half_angle
  ELSE
    step = lean_search_step, signed to match the lean's direction
    FOR angle FROM 0 STEPPING step WHILE |angle| < |half_angle|
      IF blocked_at(angle) THEN grant angle; BREAK
    # falling off the loop grants the full request

  roll = granted * 2; target_roll = roll
```

**Notes**

- Testing the **near plane's box rather than a point** is the whole reason this is not a
  ray cast: the near plane has real extent, and a lean that puts only the corner of the
  near plane into a wall still shows the player through it. The box's size is derived from
  the field of view and aspect ratio each frame, so a wider field of view leans less far —
  a real and observable coupling.
- Geometry whose material is flagged **passable** is ignored, so the player can lean
  through foliage and fences the body can also walk through.
- The sweep is linear from zero rather than a binary search. At the configured resolution
  that is up to a thousand box queries in the worst case, which is why the fast path
  exists and why the whole thing runs only for the controlled actor.
- The box's orientation is derived from the lean angle itself, tilting with the eye. A
  rebuild that tests an axis-aligned box will grant leans the original refuses.

## `cam_SetLadder` · `camUpdateLadder` · `cam_UnsetLadder`

**Contract** — while the actor is on a ladder the first-person camera's yaw is clamped to
a cone centred on the ladder's facing. Setting the clamp immediately is only safe when the
view already points roughly along the ladder; otherwise the per-frame update *rotates* the
view toward the ladder's facing at a rate proportional to the frame time, and installs the
clamp only once the view is within a small threshold. Unsetting clears the clamp.

**Invariants** — the clamp is installed exactly once per ladder occupancy, and installing
it is what makes the per-frame update a no-op thereafter.

```text
FUNCTION update_ladder(dt)
  IF not on a ladder THEN RETURN
  IF the clamp is already installed THEN RETURN

  delta = signed angle from camera yaw to ladder facing
  IF |delta| < 0.05 rad THEN
    install clamp: [ladder facing - limit, ladder facing + limit]
  ELSE
    camera yaw += delta * min(dt * 10, 1)      # ease toward the ladder, never overshoot

  IF climbing DOWN AND the camera is pitched above the downward limit THEN
    ease the pitch down toward that limit at the same rate
```

**Notes** — the easing coefficient is clamped at one so that a long frame snaps rather than
overshooting, which is the standard guard for this kind of exponential approach and worth
keeping. Forcing the pitch down while descending is a usability decision: a player
climbing down a ladder while looking up sees nothing useful and cannot judge the bottom.

## `CameraHeight`

**Contract** — the eye height above the actor's origin: a configured fraction of the
collision box's height, less a small inset. Because it is derived from the *box*, it
changes automatically when the actor crouches or the box is swapped, which is what the
smoothing above exists to hide.

## `update_camera`

**Contract** — lets the weapon-recoil effector move the view. Applies the effector's pitch
and yaw deltas to the first-person camera, re-clamping afterwards against whatever limits
are installed, and removes the effector from the stack once it reports itself finished.

**Invariants** — before the effector runs, a pitch outside its limits is brought back into
range by whole turns rather than by clamping. That preserves the *relative* offset the
recoil intends to add; clamping first would silently eat it.

**Notes** — recoil is applied to the camera, not to the weapon, and the weapon's aim is
read from the camera. That is why recoil moves the shot and not merely the picture.

## `OnRender` (debug build only)

**Contract** — draws the active item's own debug geometry, the actor's movement capsule,
and the network-prediction overlay. Compiled out of a shipping build entirely.
