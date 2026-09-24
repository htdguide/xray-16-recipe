# src/xrPhysics/PHInterpolation.cpp

> Blends a body's last two solved placements by the accumulator's leftover time, so a fixed-rate simulation renders smoothly at any frame rate.

**Needs** — [`PHInterpolation.h`](PHInterpolation.h.md) · [`PHWorld.h`](PHWorld.h.md) · [`PhysicsCommon.h`](PhysicsCommon.h.md) · [`MathUtils.h`](MathUtils.h.md) · [Seam: Rigid-body dynamics](../../SYSTEM-REQUIREMENTS.md#seam-rigid-body-dynamics)
**Used by** — [`PHInterpolation.h`](PHInterpolation.h.md)
**Tier floor** — T2: reads a body's placement and blends; the only layout concern is the quaternion convention.

## Purpose

The simulation advances in whole fixed steps; the renderer draws at whatever rate it manages. If
the renderer drew the latest solved state, objects would appear to move in the simulation's steps
rather than smoothly. This file keeps the last two solved placements and reports the point between
them corresponding to the fraction of a step the renderer is currently inside.

## State

```text
RECORD Interpolation
  body       : handle to the simulated body
  positions  : ring of 2 points        # [0] is the older sample, [1] the newer
  rotations  : ring of 2 orientations

  # invariant: both samples are always valid; binding a body fills both with its current state
  # invariant: the samples are pushed exactly once per step, from the read-back phase
```

## `set_body` / `reset_positions` / `reset_rotations`

**Contract** — bind a body and fill both samples with its current placement, so the next blend
returns that placement regardless of the blend fraction. The reset pair is the same operation
without rebinding, and is called after every teleport, every network correction and every
re-placement from animation — anywhere the body's motion between the samples would be a lie.

## `update_positions` / `update_rotations`

**Contract** — push the body's current placement as the newer sample, retiring the older one.
Called from a body's post-solve read-back, once per step.

**Notes** — the orientation is stored with its **scalar part negated** relative to the dynamics
library's convention. The two libraries here (the engine's math layer and the dynamics library)
disagree on the sign convention for a rotation quaternion, and this negation is the conversion. It
appears identically at every crossing point; a rebuild that picks one convention deletes all of
them, but must delete *all* of them, since a single missed site produces a body that rotates the
wrong way.

## `interpolate_position` / `interpolate_rotation`

**Contract** — return the placement at the current sub-step phase. Linear for position, spherical
for orientation. The phase comes from the world's accumulator remainder.

```text
FUNCTION interpolate_position(OUT point)
  t := world.frame_time / fixed_step          # in [0, 1)
  point := lerp(positions[0], positions[1], t)

FUNCTION interpolate_rotation(OUT frame)
  t := world.frame_time / fixed_step
  REQUIRE 0 <= t <= 1
  frame := matrix_of(slerp(rotations[0], rotations[1], t))
```

**Invariants** — `t` is in `[0, 1)` by the accumulator's own invariant (see
[`PHWorld.cpp`](PHWorld.cpp.md)); the assertion here is a cross-check that the world's remainder
was not left in an inconsistent state by a mid-frame step-size change.

**Notes** — spherical interpolation rather than linear is not optional for orientation. A ragdoll
limb rotating fast between steps interpolated linearly shortens visibly at the midpoint, which
reads as the limb telescoping.
