# src/xrGame/EffectorZoomInertion.cpp

> Aim sway: while a scoped weapon is held steady, the aim point wanders along a random walk whose radius and speed scale with the weapon's current dispersion — and the sway stops the moment the player moves the aim themselves.

**Needs** — [`EffectorZoomInertion.h`](EffectorZoomInertion.h.md) · [`WeaponMagazined.h`](WeaponMagazined.h.md) · [`CameraEffector.h`](CameraEffector.h.md) · [`xrEngine/CameraManager.h`](../xrEngine/CameraManager.h.md)
**Used by** — reached through its declarations in [`EffectorZoomInertion.h`](EffectorZoomInertion.h.md); callers name that, not this file.
**Tier floor** — T2: a two-dimensional random walk interpolated against the frame clock

## Purpose

The reason a scoped shot is hard to line up. A permanent camera effector that, while the
player is aiming down sights, adds a slowly wandering offset to the view direction. The
wander is a random walk in the plane: every fixed interval a new target point is drawn
inside a disc, and the offset interpolates linearly from the previous target to the new
one, so the motion is continuous with direction changes at the interval boundaries rather
than a jitter.

Two decisions shape the feel:

- **radius and speed both scale with the weapon's dispersion**, floored at configured
  minimums. A steady, accurate weapon sways in a small tight circle; an inaccurate or
  fatigued one sways wide and fast. The same number that spreads the bullets spreads the
  sight picture, so the two are legible as one property;
- **the sway is suppressed while the player is turning.** If the incoming view direction
  differs from last frame's by more than a small threshold, the offset is computed and
  advanced but not applied. The sway is a property of *holding still*, so it must never
  fight a deliberate aim adjustment.

## State

```text
RECORD ZoomInertionConfig            # per weapon, with a global section as fallback
  camera_move_epsilon : real   # how much view rotation counts as "the player is aiming"
  disp_min            : real   # floor on the wander radius
  speed_min           : real   # floor on the wander speed
  zoom_aim_disp_k     : real   # dispersion -> radius
  zoom_aim_speed_k    : real   # dispersion -> speed
  delta_time          : int    # milliseconds per leg of the walk

RECORD ZoomInertionState
  disp_radius     : real     # current wander radius
  float_speed     : real     # current wander speed
  current_point   : vector   # the offset applied this frame
  last_point      : vector   # the leg's start
  target_point    : vector   # the leg's end, drawn at random
  old_camera_dir  : vector   # last frame's view direction, for the movement test
  time_passed     : int      # milliseconds into the current leg
```

Invariants: `current_point` always lies on the segment between `last_point` and
`target_point`, so the offset is continuous across a leg boundary — the new leg starts
exactly where the old one ended. Both scaled parameters are floored, so the sway never
stops entirely however accurate the weapon.

## `LoadParams`

**Contract** — reads the six parameters, each first from a caller-supplied section under a
caller-supplied key prefix and, when that key is absent, from the global zoom-inertion
section. This two-level lookup is the whole configuration story: a weapon that wants its
own sway declares prefixed keys in its own section and overrides exactly the parameters it
names, inheriting the rest.

**Notes** — the prefix mechanism, rather than section inheritance, is used because a
weapon's section already inherits from a weapon base and cannot also inherit from an
effector section. A rebuild with multiple inheritance in its configuration format can
delete the prefix convention.

## `Load`

**Contract** — reads the global parameters with no prefix, then resets the walk: both
scaled parameters to their floors, every point to the origin, the leg clock to zero.
Establishes the state the effector runs in before any weapon has configured it.

## `Init`

**Contract** — re-reads the parameters from a specific weapon's section under the weapon
prefix. Called when the player starts aiming with that weapon. A null weapon is ignored
silently. Note that it does **not** reset the walk — switching weapons changes the
parameters mid-leg and the walk carries on from where it was, which is invisible because
the next leg boundary re-draws everything anyway.

## `SetParams`

**Contract** — pushes the weapon's current dispersion in and derives the radius and speed
from it, each floored. If the radius actually changed, the current leg is cut short so the
new radius takes effect immediately rather than after the remainder of a leg drawn at the
old one.

```text
FUNCTION set_params(dispersion)
  old_radius  = disp_radius
  disp_radius = max(dispersion * zoom_aim_disp_k,  disp_min)
  float_speed = max(dispersion * zoom_aim_speed_k, speed_min)
  IF old_radius != disp_radius THEN
    force the current leg to end on the next tick
```

**Notes** — dispersion changes every frame as the character breathes, moves and recovers,
so in practice the leg is re-drawn constantly while anything is changing and settles into
clean fixed-length legs only when the character is completely still.

## `ProcessCam`

**Contract** — the per-frame step. Decides whether the player is turning, advances the leg
clock, re-draws the target whenever a leg completes, interpolates the offset, and adds it
to the view direction only if the player is holding still. Always answers "keep me".

```text
FUNCTION process_cam(camera)
  player_turning = NOT camera.d is within camera_move_epsilon OF old_camera_dir

  IF time_passed is zero THEN
    last_point = current_point          # first leg starts wherever we already are
    draw_new_target()
  ELSE
    WHILE time_passed > delta_time      # catch up if a frame was long
      time_passed = time_passed - delta_time
      last_point  = target_point
      draw_new_target()

  current_point = interpolate(last_point, target_point, time_passed / delta_time)
  old_camera_dir = camera.d

  IF NOT player_turning THEN
    camera.d = camera.d + current_point

  time_passed = time_passed + frame_milliseconds
  RETURN keep
```

**Invariants** — the offset is advanced whether or not it is applied, so releasing the aim
and re-steadying resumes the walk mid-stride rather than snapping.

**Notes**

- The new target is drawn uniformly in a **square** of half the configured radius, not in
  a disc of that radius, so the sway is very slightly corner-biased and its actual extent
  is half the parameter's name suggests. Both are harmless and both must be reproduced to
  match the shipped tuning.
- The offset is added to the view *direction* vector, not applied as a rotation, so its
  angular effect shrinks with the vector's length. The direction is unit-length here, so
  the offset is an angle in radians to first order, which is why the configured radii are
  small numbers.
- The direction is perturbed but not renormalized. Later effectors and the camera
  itself tolerate this; a rebuild that assumes a unit direction downstream must
  renormalize.
- `draw_new_target` also records the leg's velocity vector, which nothing reads. And the
  epsilon field it sets is written at three sites and never consulted, the original
  gate having been replaced by the leg clock.

## Could not recover

- `m_fEpsilon` is maintained carefully — set to twice the speed at each leg boundary and
  to twice the radius when the radius changes — and read nowhere. The "force the leg to
  end" effect described under `SetParams` is therefore *intended* but not actually
  achieved; the leg simply runs to its normal end. Reproducing the original's behaviour
  means reproducing the dead write, not the intent.
- `m_vTargetVel` is computed per leg and never used.
