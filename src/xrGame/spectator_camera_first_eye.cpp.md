# src/xrGame/spectator_camera_first_eye.cpp

> A first-person camera whose look speed is scaled by a frame time supplied from outside, so that a spectator's free look advances at the same rate whether or not the simulation it is watching is running.

**Needs** — [`spectator_camera_first_eye.h`](spectator_camera_first_eye.h.md) · [`CameraFirstEye.h`](CameraFirstEye.h.md) · [`xrEngine/xr_level_controller.h`](../xrEngine/xr_level_controller.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: angle arithmetic and two clamps

## Purpose

The ordinary first-person camera derives its turn rate from the global frame time. A
spectator watching a multiplayer match is not attached to a simulated body and may be
looking around while the thing it is looking at is paused, dead or being replaced, so its
turn rate has to come from a clock the spectator controller chooses. This subclass changes
exactly that one thing.

It exists as a separate type rather than a flag on the base camera because the substitute
clock is a *reference* to a value the owner updates, not a value passed per call — the
camera reads whatever the spectator controller last wrote.

## State

```text
RECORD SpectatorFirstEyeCamera        # extends the first-person camera
  frame_time : reference to real      # NOT owned; the spectator controller's own delta
```

**Invariants** — the referenced value must outlive the camera. It is a field of the
spectator controller that constructs it, which is what makes that safe. Copying a camera is
forbidden, because a copy would share the reference without any guarantee about the
original's lifetime.

## `Move`

**Contract** — apply one look command: pitch up or down, yaw left or right. Takes an
explicit angular step, or zero to mean "derive it from the configured turn speed and the
supplied frame time". Takes a divisor that slows the derived step — how a zoomed or scoped
view turns more slowly. Mutates the camera's angles in place.

```text
FUNCTION Move(command, step, slowdown)
  # Bring pitch into the clamp range by whole turns first, so that a pitch that has
  # wrapped past a full revolution clamps to the near limit rather than the far one.
  IF pitch is clamped
    WHILE pitch < pitch_min   pitch = pitch + one full turn
    WHILE pitch > pitch_max   pitch = pitch - one full turn

  delta = step IF step is non-zero
          ELSE turn_speed_for_the_axis * frame_time / slowdown

  SELECT command
    down  -> pitch = pitch - delta
    up    -> pitch = pitch + delta
    left  -> yaw   = yaw   - delta
    right -> yaw   = yaw   + delta

  IF yaw is clamped    clamp yaw into its limits
  IF pitch is clamped  clamp pitch into its limits
```

**Invariants** — the whole-turn normalization runs **before** the step, not after. Clamping
a pitch that is several revolutions out would otherwise snap it to a limit and lose the
direction the player was looking; normalizing first means the clamp only ever has to
correct the current step.

**Notes** — a step of zero standing for "use the configured rate" conflates "no movement"
with "default movement", which is inherited from the base camera's interface and is
harmless only because nothing ever asks for a genuinely zero step. A rebuild should use an
absent value instead.

The two axes read separate configured turn speeds, so horizontal and vertical sensitivity
are independently tunable — which they must be, because the vertical range is a fraction of
the horizontal one.

## Constructor and destructor

**Contract** — construction binds the frame-time reference and forwards the owning object
and the camera flags to the base. Nothing else is set up; every other camera parameter is
the base's.
