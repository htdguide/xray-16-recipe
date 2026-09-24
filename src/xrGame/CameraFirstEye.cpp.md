# src/xrGame/CameraFirstEye.cpp

> The first-person camera: the eye sits exactly where it is put, looks where the player aims, and can be eased onto a point when something else wants to direct the view.

**Needs** — [`CameraFirstEye.h`](CameraFirstEye.h.md) · [`xrEngine/CameraBase.h`](../xrEngine/CameraBase.h.md) · [`xrEngine/xr_level_controller.h`](../xrEngine/xr_level_controller.h.md)
**Used by** — reached through its declarations in [`CameraFirstEye.h`](CameraFirstEye.h.md); callers name that, not this file.
**Tier floor** — T2: per-frame rotation composition

## Purpose

The simplest of the three cameras and the one that is always updated (see
[`ActorCameras.cpp`](ActorCameras.cpp.md)). Its position is whatever the owner computed;
its orientation is the yaw and pitch the player has accumulated, composed with a
per-frame noise offset the effector stack supplies.

Its one non-obvious feature is the **look-at**: a caller can name a world point and the
camera eases onto it, disengaging by itself once it has arrived. That is how a script turns
the player's head toward something without taking the camera away entirely.

## State

```text
  look_at_point  : vector    # the point to turn toward
  look_at_active : bool      # cleared automatically on arrival
  yaw, pitch, roll           # inherited: the accumulated aim
```

## `Update`

**Contract** — places the camera at the supplied point, advances any active look-at, and
builds the view basis from the aim angles composed with a noise offset.

```text
FUNCTION update(point, noise_angles)
  position = point
  advance_lookat()

  noise = rotation from the three noise angles, applied in X then Y then Z order
  aim   = rotation from (roll, yaw, pitch), TRANSPOSED
  basis = aim composed with noise
  direction = basis forward; up = basis up

  IF this camera is linked to its parent's frame THEN
    rotate both vectors by the parent's transform
```

**Invariants** — the aim rotation is transposed rather than inverted, which is the same
thing for a pure rotation and cheaper. The noise offset's yaw is negated to match the
camera's sign convention, which is the opposite of the skeleton's; the chapter has at least
three such conventions meeting, and a rebuild should pick one.

**Notes** — the relative-link flag decides whether this camera's orientation is in world
space or in its owner's. The actor's is world-space; a camera mounted in a vehicle is not.

## `Move`

**Contract** — the rotation input. Each of the four directions adds either an explicitly
supplied amount or, when none is given, a rate-times-frame-time step divided by the caller's
inertia factor. Both angles are clamped afterwards if limits are installed.

**Invariants** — before moving, a clamped pitch is brought into its limit range **by whole
turns** rather than by clamping, so that an angle that wrapped past the limit is restored
rather than pinned. The same trick appears in the recoil path
([`ActorCameras.cpp`](ActorCameras.cpp.md)) and for the same reason.

**Notes** — the zero-means-default convention on the amount makes a deliberate
zero-magnitude move impossible. Harmless here, since a zero move is a no-op anyway.

## `OnActivate`

**Contract** — inherits the outgoing camera's yaw, but only when both cameras use the same
space (both world-relative, or both parent-relative). Mixing the two would inherit an angle
that means something different.

**Notes** — pitch is not inherited, only yaw. Switching to first person from a
third-person view therefore preserves which way you face and resets how far up or down you
look.

## the look-at easing (private)

**Contract** — while active, computes the yaw and pitch that would point at the stored point
from the camera's current position, and eases both toward it with an inertial approach.
Deactivates itself once both are within a small tolerance.

**Invariants** — the target angles are recomputed every frame from the *current* position,
so the easing tracks a point correctly even while the player is moving.

**Notes** — the arrival test runs *before* the easing step, so the camera always takes one
extra frame after arriving. Immaterial, but a rebuild ordering it the other way will differ
by a frame.
