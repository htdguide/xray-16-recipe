# src/xrGame/EffectorBobbing.cpp

> The walk cycle's effect on the first-person camera: a figure-of-eight bob whose amplitude and rate follow the gait, faded in and out so that starting and stopping do not snap.

**Needs** — [`EffectorBobbing.h`](EffectorBobbing.h.md) · [`Actor.h`](Actor.h.md) · [`actor_defs.h`](actor_defs.h.md) · [`CameraEffector.h`](CameraEffector.h.md) · [`xrEngine/CameraManager.h`](../xrEngine/CameraManager.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: a trigonometric perturbation of a camera basis, every frame

## Purpose

A camera effector — one entry in the chain the camera manager runs each frame, each
handed the camera's position and basis vectors and free to perturb them. This one adds
head bob. It is a permanent effector installed for the lifetime of the player character
rather than a transient one, which is why its declared lifetime is effectively infinite;
it is silenced by fading its own strength to zero, not by expiring.

Three decisions carry the file:

- the bob's amplitude and rate come from a named configuration section, not from code, in
  three gait pairs (run, walk, limp), so the feel is tunable without a build;
- the phase clock runs continuously even while standing still, so that stopping and
  starting again resumes mid-stride rather than snapping to the start of a step;
- the strength ramps in and out over a fixed time rather than switching, which is what
  makes the previous point invisible.

## State

```text
RECORD BobbingConfig                # read once at construction from the bobbing section
  amplitude_run   : real
  amplitude_walk  : real
  amplitude_limp  : real
  speed_run       : real            # radians of phase per second
  speed_walk      : real
  speed_limp      : real

RECORD BobbingState
  phase_time       : real   # seconds accumulated; never reset, so the cycle is continuous
  strength         : real   # in [0,1]; the fade-in/out factor
  movement_flags   : int    # the character's current movement bitset, pushed in by the owner
  limping          : bool
  zoom_mode        : bool   # aiming down sights; changes what counts as "accelerated"
```

Invariants: `strength` is clamped to `[0,1]`; when it is zero the effector leaves the
camera untouched and costs nothing but the clock update. `phase_time` grows without bound
over a session — see Notes.

## `SetState`

**Contract** — the owner pushes in the character's movement bitset, whether the character
is limping from injury, and whether it is aiming. Called every frame before the camera
chain runs; the effector never reaches back to the character to read these.

**Notes** — pushing rather than pulling is what keeps the effector usable by anything with
a movement state, not just the player character. It is the only coupling the effector has
to the game layer.

## `ProcessCam`

**Contract** — the per-frame camera perturbation. Advances the phase clock, moves the
strength toward one while any movement flag is set and toward zero otherwise at a fixed
rate, and, if the strength is non-zero, raises the camera and rolls its basis by a pair of
sinusoids of the current gait's amplitude and rate. Always answers "keep me" — the
effector never expires.

```text
FUNCTION process_cam(camera)         # camera has position p, direction d, up n
  phase_time = phase_time + frame_seconds

  IF any movement flag set THEN
    strength = min(strength + fade_rate * frame_seconds, 1)
  ELSE
    strength = max(strength - fade_rate * frame_seconds, 0)

  IF strength is zero THEN RETURN keep

  crouch_scale = 0.75 IF crouching ELSE 1

  IF accelerated(movement_flags, zoom_mode) THEN
    amplitude, rate = amplitude_run, speed_run
  ELSE IF limping THEN
    amplitude, rate = amplitude_limp, speed_limp
  ELSE
    amplitude, rate = amplitude_walk, speed_walk

  amplitude = amplitude * crouch_scale
  angle     = rate * phase_time * crouch_scale

  rise = |sin(angle)| * amplitude * strength    # absolute: two rises per cycle, one per foot
  sway =  cos(angle)  * amplitude * strength    # signed: one sway per cycle, alternating feet

  camera.p.height = camera.p.height + rise
  rotate the camera basis by (heading: sway, pitch: rise, bank: sway)
  RETURN keep
```

**Invariants** — the rotation is applied to the basis built from the *incoming* direction
and up vectors, so effectors earlier in the chain compose correctly and this one never
assumes it runs first.

**Notes**

- The absolute value on the vertical term is the load-bearing shape decision. It makes the
  head rise twice per cycle — once per footfall — while the lateral sway alternates once
  per cycle, which is what reads as walking rather than as bobbing on a spring. A rebuild
  that drops the absolute value gets a visibly wrong gait.
- Scaling *both* amplitude and rate by the crouch factor slows the cycle as well as
  shrinking it, so crouching reads as a different gait rather than the same gait made
  small.
- Gait selection asks the character's own "is this an accelerated move" predicate, which
  folds in sprinting, aiming and the movement bitset together; aiming suppresses the run
  gait even while running, which is why the zoom flag is pushed in at all.
- The phase clock accumulates for the whole session and is never wrapped. At single
  precision this loses phase resolution after some hours of play; nothing depends on the
  phase being exact, so the original does not care. A rebuild should wrap it anyway.
- The fade rate and the crouch factor are compiled-in constants while every other number
  is configuration. That asymmetry looks unintentional.
