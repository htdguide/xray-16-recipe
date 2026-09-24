# src/xrGame/poses_blending.cpp

> Moving smoothly from one rigid pose to another: rotation on the shortest arc, position in a straight line, over a fixed duration.

**Needs** — [`poses_blending.h`](poses_blending.h.md)
**Used by** — [`poses_blending.h`](poses_blending.h.md)
**Tier floor** — T3: two-pose interpolation

## Purpose

When something's transform must be handed from one authority to another — an animation
taking over from physics, a camera leaving a scripted position — snapping is visible and
blending is not. This is the blend.

The decision worth recording is that **rotation and position are interpolated separately and
differently**. Interpolating the transform's entries directly would shear and scale the
result mid-way; splitting it into an orientation and a point, running the orientation along
the shortest arc between the two and the point along the straight line between them, and
recombining, produces the only motion that looks like a rigid body moving.

## State

```text
RECORD PosesInterpolation
  p0, p1 : (real, real, real)      # start and end positions
  q0, q1 : orientation             # start and end orientations

RECORD PosesBlending
  interpolation : PosesInterpolation
  target_time   : real             # the duration of the trip
```

**Invariants**
- Both are captured at construction and immutable afterwards. A blend whose endpoints move
  is a different mechanism and this is not it.
- The duration is strictly positive.
- The blend is queried only with a time within the trip. Reaching or passing the duration is
  the caller's signal to stop asking, not a case this handles.

## `poses_interpolation`

**Contract** — construct from two transforms, decomposing each into a position and an
orientation. Then, for a fraction, produce the transform between them. The fraction is not
clamped: a caller outside zero to one gets extrapolation, and nothing here treats that as an
error.

```text
FUNCTION pose(out transform, factor)
  transform.rotation = shortest_arc_interpolate(q0, q1, factor)
  transform.position = linear_interpolate(p0, p1, factor)
```

**Notes** — shortest-arc interpolation is what makes a blend between two orientations more
than a hundred and eighty degrees apart take the near way round. The alternative — treating
the orientation as four numbers and interpolating them — both takes the wrong path and
changes speed through the turn.

The result is rebuilt from scratch each call rather than accumulated, so repeated queries at
the same fraction give identical answers and no drift accumulates over a long blend.

## `poses_blending`

**Contract** — the interpolation plus a duration. Queried with an elapsed time rather than a
fraction; converts and delegates. Reports separately whether the duration has elapsed.

```text
FUNCTION pose(out transform, time)
  interpolation.pose(transform, time / target_time)

FUNCTION target_reached(time) -> bool
  RETURN time >= target_time
```

**Notes** — the completion test is a separate question rather than a return value from the
pose query, which is what lets a caller drive the blend from its own clock and decide what
happens at the end — hold the final pose, release the transform to its new owner, or start
another blend. The blend itself has no terminal state.

Time is measured from the start of the blend by the caller. Nothing here holds a clock,
which is what makes the type usable from a frame loop, from a fixed-timestep simulation
step, and from a scrubbed replay without change.
