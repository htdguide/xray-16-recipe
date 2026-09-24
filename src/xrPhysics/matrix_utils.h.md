# src/xrPhysics/matrix_utils.h

> Compares, clamps and differentiates rigid transforms — the small algebra that every
> animation-to-physics blend is built out of.

**Needs** — [`xrMiscMath`](../utils/xrMiscMath/README.md)
**Used by** — [`animation_movement_controller.cpp`](../xrGame/animation_movement_controller.cpp.md) · [`moving_bones_snd_player.cpp`](../xrGame/moving_bones_snd_player.cpp.md) · [`ActorCameraCollision.cpp`](ActorCameraCollision.cpp.md) · [`PHElement.cpp`](PHElement.cpp.md)
**Tier floor** — T2: pure transform algebra. Nothing here touches a device or a layout; it is
T1 in the original only because it is inlined into hot paths.

## Purpose

Header-only. Six small operations, and every one of them exists to answer a question the
ragdoll and vehicle blending code keeps asking: *how far apart are these two poses, and how do
I move from one to the other without jumping?*

They are grouped here rather than in the general math layer because they share one convention
— a **difference transform**, the transform that takes one pose to another — and because the
clamping ones encode a policy (what counts as "too far"), not just arithmetic.

## Stateless.

## `clamp_rotation`

**Contract** — limit a rotation to a maximum angle, and report the angle it had. Given a
rotation and a cap, if the rotation's magnitude exceeds the cap it is replaced by a rotation
about the *same axis* at exactly the cap, preserving the sign. Returns the original magnitude,
so a caller can both clamp and test in one call.

```text
FUNCTION clamp_rotation(rotation, cap) -> original_magnitude
  (axis, angle) := axis_and_angle_of(rotation)
  IF |angle| > cap
    rotation := rotation_about(axis, cap with the sign of angle)
  RETURN |angle|
```

A second form takes a full transform: it clamps the rotational part and leaves the translation
untouched.

**Invariants** — the axis survives the clamp. Clamping by scaling the quaternion's components
or by interpolating toward identity would change the axis and make a limited rotation drift
sideways; taking the axis out, capping the angle, and rebuilding is what keeps the direction.

## `clamp_change`

**Contract** — given a current pose, a starting pose, a linear and an angular *clamp*, and a
linear and an angular *tolerance*, restrict how far the current pose may have moved from the
start, and report whether it is within tolerance.

```text
FUNCTION clamp_change(inout pose, start, max_linear, max_angular,
                      tol_linear, tol_angular) -> within_tolerance
  diff := start⁻¹ ∘ pose              # the movement, in the start's frame
  moved := |diff.translation|
  within := moved < tol_linear

  IF moved > max_linear
    diff.translation := diff.translation scaled to length max_linear
  IF clamp_rotation(diff, max_angular) > tol_angular
    within := false

  IF NOT within
    pose := start ∘ diff              # write the clamped pose back
  RETURN within
```

**Invariants** — two thresholds per axis of freedom, and they are not the same number. The
**clamp** is how far the pose is *allowed* to be; the **tolerance** is how close it must be to
count as arrived. Clamp is always the larger. Collapsing them into one number gives either a
blend that snaps (clamp too small) or one that never terminates (tolerance too large).

The pose is written back only when it is out of tolerance. That means a pose already within
tolerance is left bit-identical, which matters for
[determinism](../../SYSTEM-REQUIREMENTS.md#6-conformance): a no-op must actually be a no-op.

**Notes** — this is the whole of the blending policy for animation-driven physics. A body
being dragged toward an animated pose is clamped each step so it cannot cross the world in one
frame, and the returned flag is the blend's exit condition.

## `get_diff_value` / `cmp_matrix`

**Contract** — `get_diff_value` reports the linear and angular distance between two poses as
two non-negative scalars. `cmp_matrix` compares those against a pair of tolerances, either
reporting the two verdicts separately or a single conjunction.

**Notes** — the separate-verdicts form exists because a blend often finishes in position long
before it finishes in orientation, and the two halves are driven by different controllers.

## `angular_diff`, `linear_diff`, `matrix_diff`

**Contract** — convert a difference transform and a time interval into the velocities that
would have produced it.

```text
FUNCTION matrix_diff(from, to, dt) -> (linear_velocity, angular_velocity)
  diff := from⁻¹ ∘ to
  linear  := diff.translation / dt
  angular := ( (diff[2][1] - diff[1][2]) / 2,
               (diff[0][2] - diff[2][0]) / 2,
               (diff[1][0] - diff[0][1]) / 2 ) / dt
```

**Invariants** — the angular part is the **antisymmetric part of the rotation matrix**, which
is a small-angle approximation of the rotation vector. It is exact only in the limit and
degrades as the rotation approaches a half turn — beyond a quarter turn per step it starts
under-reporting, and at a half turn it reports zero. That is acceptable here because it is
only ever applied to one fixed step's worth of animation motion, which is small by
construction. A rebuild that reuses it for larger intervals must switch to the axis-and-angle
form.

**Notes** — this is the bridge in the other direction from `clamp_change`: that one drags a
body toward a pose, this one converts a pose change into the velocity to *give* a body so it
arrives there on its own. Both are needed, because a ragdoll being released from animation
must inherit the animation's velocity or it drops as if it had been standing still.
