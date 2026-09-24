# src/xrGame/matrix_utils.h

> Comparing, limiting and differentiating rigid transforms: the arithmetic behind "has this bone moved enough to matter" and "how fast is it moving".

**Needs** — _none beyond the engine's transform and rotation types_
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T1: per-frame transform arithmetic in the animation and physics path

## Purpose

Animation and physics constantly need to ask three questions about a pair of rigid
transforms: are they close enough to treat as equal, how far apart are they, and what
velocity would carry one to the other in a frame. Each question separates into a *linear*
and an *angular* part with independent tolerances, because a millimetre and a degree are not
comparable quantities and no single number can stand for both.

This file is that arithmetic, in one place, with no state. It is a header with no
implementation because every entry is a handful of operations on a caller's transform.

## State

`Stateless.`

**Invariant** — every function here treats a transform as *rigid*: rotation plus
translation, no scale. The difference between two transforms is computed as the inverse of
one composed with the other, which is only a meaningful "difference" for rigid transforms.
A scaled transform passed in produces a difference whose rotation part is not a rotation, and
every downstream extraction is then wrong.

## `get_diff_value`

**Contract** — reports how far apart two transforms are, as two independent numbers: the
distance between their origins and the magnitude of the rotation between them, in radians.
The angle is returned unsigned, because "how different" has no direction.

```text
FUNCTION get_diff_value(a, b) -> (distance, angle)
  diff     = inverse(b) COMPOSED WITH a     # a expressed in b's frame
  distance = magnitude of diff's translation
  angle    = |rotation angle of diff|
```

**Invariant** — the difference is expressed in the *second* transform's frame. Because the
two quantities extracted are a magnitude and an angle, both of which are invariant under a
change of frame, the choice does not affect the answer — but it does fix the sign convention
that `clamp_change` relies on.

## `cmp_matrix`

**Contract** — two forms. One reports the linear and angular comparisons separately as two
booleans; the other combines them, answering true only when both are within tolerance.
Tolerances are always supplied by the caller — there is no default — because what counts as
"the same place" differs by orders of magnitude between a weapon's muzzle and a vehicle.

**Notes** — separating the two results is not a convenience. Code that synchronizes a
physics body to an animated bone commonly needs to correct position without correcting
orientation, or the reverse, and needs to know which one drifted.

## `clamp_rotation`

**Contract** — limits a rotation to a maximum angle, in place, and returns the angle it
*had*. Two forms: one on a rotation alone, one on a full transform, where the translation is
preserved unchanged across the clamp. A rotation already within the limit is left exactly as
it was; one beyond it is replaced by a rotation of the limit magnitude **about the same
axis**, with the original's sign.

```text
FUNCTION clamp_rotation(INOUT rotation, limit) -> original_angle
  (axis, angle) = axis and angle of rotation
  IF |angle| > limit THEN
    rotation = rotation of (limit, signed as angle was) about axis
    renormalize
  RETURN |angle|
```

**Invariants** — the axis is preserved and only the magnitude is limited, so a clamped
rotation still turns the *right way*; clamping toward identity instead would produce visible
snapping. Renormalization after rebuilding the rotation is not optional: the result feeds
further composition, and drift accumulates over a frame chain.

Returning the original angle rather than the clamped one is what lets a caller detect that
clamping happened and by how much, in the same call.

## `clamp_change`

**Contract** — limits how far a transform may have moved from a reference, and reports
whether the movement was small enough to be treated as insignificant. Takes two limits (the
most linear and angular change permitted) and two thresholds (below which the change is
"nothing happened"). The transform is modified in place **only when the change is
significant**.

```text
FUNCTION clamp_change(INOUT m, reference, max_linear, max_angular,
                      tol_linear, tol_angular) -> unchanged
  diff = inverse(reference) COMPOSED WITH m
  moved = magnitude of diff's translation
  unchanged = moved < tol_linear

  IF moved > max_linear THEN scale diff's translation down to max_linear
  IF clamp_rotation(diff, max_angular) > tol_angular THEN unchanged = false

  IF NOT unchanged THEN m = reference COMPOSED WITH diff
  RETURN unchanged
```

**Invariants** — the limits and the thresholds are different things and both are needed. The
*limit* caps how far the transform is allowed to jump in one step, which is what stops a
physics correction from teleporting a bone. The *threshold* decides whether the step is worth
propagating at all, which is what stops a chain of tiny corrections from costing work every
frame.

The ordering is load-bearing: the linear result is computed **before** the clamp is applied,
so the answer describes the movement that was *requested*, not the movement that was allowed.
A caller whose request was clamped is told the change was significant, which is correct — it
was, it was just limited.

Writing back only when significant means an insignificant change leaves the transform exactly
as the caller had it, bit for bit, rather than as a round trip through the difference. That
preserves idempotence: repeatedly clamping an already-settled transform never drifts.

## `angular_diff` · `linear_diff` · `matrix_diff`

**Contract** — convert a transform difference into the velocities that would produce it over
a time step: an angular velocity vector and a linear velocity vector. `matrix_diff` does
both, either from a ready difference or from two transforms.

The angular velocity is extracted from the **antisymmetric part** of the difference's
rotation — the three off-diagonal pairs, each halved — divided by the step. This is the
small-angle approximation: for a rotation of angle θ the antisymmetric part has magnitude
sin θ rather than θ, so the result is accurate only while the per-step rotation is small.
Over one frame of a rigid body it is, which is why this cheap form is used instead of
extracting the axis and angle properly.

**Notes** — the approximation degrades toward a half turn, where it reports *zero* angular
velocity rather than the maximum. Any caller that can see rotations approaching a half turn
per step must extract the axis and angle instead. A rebuild should note the limit rather than
silently reproduce the formula.

## `get_axis_angle`

**Contract** — extracts the rotation axis and signed angle from a transform, discarding the
translation. A helper; the conversion belongs to the rotation type and is spelled out here
only because the transform type does not offer it directly.
