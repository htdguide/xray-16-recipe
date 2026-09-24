# src/xrGame/aimers_base.cpp

> The aiming solver: given a bone and something rigidly attached to it that points somewhere, find the rotation of that bone which makes the attached thing point at a target instead.

**Needs** — [`aimers_base.h`](aimers_base.h.md) · [`aimers_base_inline.h`](aimers_base_inline.h.md) · [`GameObject.h`](GameObject.h.md) · [`animation_movement_controller.h`](animation_movement_controller.h.md) · [`Include/xrRender/Kinematics.h`](../Include/xrRender/Kinematics.h.md)
**Used by** — [`aimers_base.h`](aimers_base.h.md)
**Tier floor** — T2: dense vector arithmetic on a hot path, but no byte layout and no device.

## Purpose

An authored animation makes a character raise a weapon; the animation does not know where
the target is. Aiming is therefore a *correction* applied on top of the animation: one or
more bones are rotated so that the thing at the end of the chain — a gun's muzzle, a
creature's head — lines up with a world position. This file owns the geometry of that
correction, shared by every kind of aimer.

The problem is harder than "rotate the bone to face the target" because the aiming
reference is not at the bone's origin: the muzzle sits some way off the bone, so rotating
the bone sweeps the muzzle around an arc rather than pivoting it in place.

## State

```text
RECORD Aimer                       # the base carried by every aimer
  object          : reference to the game object being aimed
  kinematics      : the skeleton interface of its visual
  animated        : the animation interface of the same visual
  target          : reference to the world position to aim at
  animation_id    : the motion whose pose the correction is computed against
  animation_start : bool    # sample the motion at its first frame, or its last
  start_transform : transform    # the object's world placement the correction is
                                 #   expressed relative to

# Invariant: target is held by reference, not copied — the aimer is a short-lived
#   object built during one animation-selection decision and the caller owns the point.
# Invariant: start_transform is the movement controller's start transform when the
#   object is under animation-driven movement, and the object's current placement
#   otherwise. Using the current placement during a root-motion animation would fold
#   the animation's own displacement into the aiming correction.
```

## `aim_at_position`

**Contract** — Computes the rotation to apply at a bone so that a ray originating at a
given point with a given direction — both rigidly attached to that bone — passes through
the target. Pure geometry: reads no engine state beyond the target, writes only its result.
Every intermediate is guarded against degeneracy; none of the guards can fail the call,
they all substitute a safe value.

The insight is that rotating about the bone's origin preserves the ray's *perpendicular
distance* from that origin. So the ray, after any rotation, remains tangent to a sphere
centred on the bone with that radius — and a ray through the target tangent to that sphere
touches it somewhere on a known circle.

```text
FUNCTION aim_at_position(bone_position, ray_origin, ray_direction) -> rotation
  normalize ray_direction
  # 1. The point on the ray closest to the bone. Its distance from the bone is the
  #    radius that rotation cannot change.
  foot   = ray_origin + ray_direction * dot(bone_position - ray_origin, ray_direction)
  arm    = foot - bone_position                  # substituted with a tiny forward
                                                 #   vector if it degenerates to zero
  radius_sq = |arm|^2

  # 2. Rays from the target tangent to that sphere touch it on one circle,
  #    perpendicular to the bone-to-target line.
  to_target = normalize(target - bone_position); L = |target - bone_position|
  centre_offset = radius_sq / L
  circle_centre = bone_position + to_target * centre_offset
  circle_radius = sqrt(max(radius_sq - centre_offset^2, 0))

  # 3. Of all points on that circle, pick the one closest to where the ray already
  #    is — the minimal correction, so the character does not spin to a mirror pose.
  projection  = foot projected onto the circle's plane
  spoke       = normalize(projection - circle_centre)
  tangency    = circle_centre + spoke * circle_radius

  # 4. Two rotations compose the answer.
  swing = rotation taking normalize(arm) to normalize(tangency - bone_position)
  twist = rotation taking (ray_direction after swing)
                       to normalize(target - tangency)
  RETURN twist composed after swing
```

**Invariants** — The two rotations are not interchangeable. The first moves the ray's
perpendicular foot onto the tangency circle; the second spins the ray about the now-correct
axis until it actually points at the target. Applying them in the other order aims a ray
whose foot has not yet moved.

The radius-squared term is clamped at zero before the square root: when the target is
closer to the bone than the muzzle offset, there is no tangent line at all and the circle
collapses to a point. The correction then aims as close as the geometry allows rather than
failing.

**Notes** — Each rotation is built from a cross product and an arc-tangent of the sine and
cosine, which is the numerically stable way to get an angle over the full half-turn range;
the cross product's magnitude *is* the sine. Three degenerate cases are handled explicitly
because the cross product vanishes for both parallel and antiparallel vectors and the two
need opposite answers:

- vectors already aligned — the identity;
- vectors opposed — a half turn, about any perpendicular axis, preferably one derived from
  the bone-to-target line so the character turns the short way;
- no perpendicular axis derivable at all — a half turn about the object's own forward axis,
  which is arbitrary but never produces an invalid transform.

A rebuild can collapse all three into a single "rotation between two unit vectors" routine
with a stable antiparallel fallback; the engine's version is spelled out because it wants
a *specific* fallback axis.

## `callback`

**Contract** — The hook the skeleton calls while evaluating one bone's world transform.
Applies a pre-computed rotation to the bone in place while *preserving the bone's
translation*, so the correction rotates the limb without displacing its joint. The rotation
is passed through the skeleton's per-bone user parameter rather than captured, because the
skeleton evaluates bones without knowing who installed the hook.

**Invariants** — Saving and restoring the position around the multiplication is the whole
function: composing the rotation on the left would otherwise swing the joint itself about
the object's origin and detach the limb.

## Construction

**Contract** — Binds the aimer to an object, resolves the named motion to a motion
identifier, records whether the correction is computed at the motion's first or last frame,
and captures the reference transform described in the state block above. Fails an assertion
if the object has no skeleton or no animation interface — an aimer is only ever built for
an animated object, so that is a caller error rather than a condition.

**Notes** — Sampling the motion at its *last* frame is the common case: the question an
aimer answers is "if he plays this animation, where will the muzzle end up pointing?", and
the answer that matters is the pose the animation finishes in. The first-frame mode exists
for the case where the animation is already blending in and the correction must match where
it currently is.
