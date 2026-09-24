# src/xrGame/detail_path_manager.cpp

> The detail path's lifecycle and its queries: build, validate, report where the follower is, how far is left, and construct the two turning circles a point's heading and speed imply.

**Needs** — [`detail_path_manager.h`](detail_path_manager.h.md) · [`ai_space.h`](ai_space.h.md) · [`xrAICore/Navigation/level_graph.h`](../xrAICore/Navigation/level_graph.h.md) · [`xrAICore/Navigation/ai_object_location.h`](../xrAICore/Navigation/ai_object_location.h.md) · [`GameObject.h`](GameObject.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: plane geometry over a navigation mesh

## Purpose

The coarse pathfinder produces a list of navigation vertices. That list is not a path a body
can follow: it is a sequence of cell identifiers with right-angle turns and no notion of
turning radius or speed. This class is the second stage — it produces the curve, sampled
into points that each carry a navigation vertex and a velocity identifier.

This file holds everything except the smoothing algorithm itself, which is in
[`detail_path_manager_smooth.cpp`](detail_path_manager_smooth.cpp.md).

## State

See [`detail_path_manager.h`](detail_path_manager.h.md) for the records. The instance state
divides into *inputs* (start and destination position and heading, the velocity masks, the
flags), the *output* (the point list plus the corrected destination and the destination
vertex), and *status*.

**Four statuses, and they are not the same question.** Getting these confused is the main
hazard of the class:

- **actual** — the path was built and none of its inputs have changed since. Set by a
  successful build; cleared by any setter that receives a different value.
- **failed** — the last build attempt did not produce a path. Independent of actuality: a
  failed build leaves the old path in place and still actual only if nothing changed.
- **valid** (no argument) — the built path actually *ends where it was asked to*.
- **valid(position)** — the position is a finite, representable point. A precondition
  check, unrelated to the path.

**Invariant** — the reset state has no current travel point (an out-of-range marker), an
empty path, the default smooth path type, an all-bits desirable mask and an empty velocity
mask, and an extrapolation length of **8** metres. The asymmetry between the two masks is
the policy: *any* velocity may be used unless the caller narrows it, but *no* velocity is
enabled until the caller says so, so a creature that never sets a velocity mask cannot move
at all rather than moving at an arbitrary speed.

## `build_path`

**Contract** — the entry point. Takes the coarse vertex list and an intermediate index into
it. Refuses to run at all if either endpoint is not a representable position. Dispatches on
the path type, marks failure, and on success adopts the result.

```text
FUNCTION build_path(level_path, intermediate_index)
  IF start or destination is not a representable position THEN RETURN
  build the smooth path            # all three path types take this branch
  IF failed THEN log the endpoints and the velocity table   # the only diagnostic
  IF valid() THEN
    actuality = true
    current_travel_point = 0
    time_path_built = now
```

**Invariants** — actuality is granted only when the path both succeeded *and* ends at the
corrected destination. A path that was built but stops short is not adopted, and the
follower keeps the previous one.

**Notes** — the three path types dispatch to the same builder. The dodge and criteria
variants were designed and never diverged; a rebuild has one path type until it needs more.

## `valid`

**Contract** — true when the path is non-empty and its **last meaningful point coincides
horizontally with the corrected destination**. Which point counts as last depends on the
patrol flag: an ordinary path is checked at its final point, a patrol path at the recorded
last-patrol index, because a patrol path deliberately extends past its destination (see
`add_patrol_point` in
[`detail_path_manager_smooth.cpp`](detail_path_manager_smooth.cpp.md)).

**Invariants** — the comparison is against the **corrected** destination, not the requested
one. The builder nudges the requested destination to a point genuinely inside a navigation
vertex, and validating against the original would reject every path whose destination sat
fractionally outside the mesh.

**Invariants** — the comparison ignores the vertical. Heights are assigned from the mesh
afterwards and need not match a caller's guess.

## `direction` · `try_get_direction`

**Contract** — the heading from the current point to the next. The two differ only in what
they do when there is no next point or the segment is degenerate: `direction` returns a
fixed unit vector along the world axis, `try_get_direction` reports failure. Callers that
would rather stop than face an arbitrary way use the second.

**Notes** — the fallback heading is a world-space constant with no relation to the
creature's facing. Any caller that uses `direction` at the end of a path snaps to it. That
is why the second form exists and why new callers should prefer it.

## `update_distance_to_target` · `distance_to_target`

**Contract** — the remaining path length, summed over every segment from the point *after*
the current one to the end. Computed lazily and cached; the cache is invalidated when the
follower advances a point, and returns zero when there is no actual path.

**Invariants** — it measures the *path*, not the straight-line distance, which is the whole
point: a creature about to walk around a building must not believe it is two metres from its
goal.

**Notes** — the sum starts from the segment ending at the point after the current one, so
the partial segment the creature is standing in is excluded. The error is at most one
sample's spacing and is never corrected.

## `on_travel_point_change`

**Contract** — invalidates the cached remaining distance. Called by the follower each time
it advances. It receives the previous index and ignores it.

## `location_on_path`

**Contract** — walks forward from the current point until the accumulated path length
exceeds the requested distance, and reports that point's position and navigation vertex.
Used to answer "where will I be in *n* metres" for look-ahead: animation selection, obstacle
anticipation, and the decision to start slowing for a corner. Falls back to the object's
current position and vertex when there is no actual path, and to the path's end when the
requested distance exceeds what is left.

**Notes** — the returned point is a *sample*, not an interpolation, so the answer is
quantized to the path's sampling. The samples are dense (see the minimum-distance constant
in [`detail_path_manager_smooth.cpp`](detail_path_manager_smooth.cpp.md)) so the error is
well under a tenth of a metre.

## `assign_angle`

**Contract** — computes the signed swept angle from a start heading to a destination heading
around a turning circle, given whether the turn is positive (counter-clockwise) and the
direction type of the two legs. Pure.

```text
FUNCTION assign_angle(start_yaw, dest_yaw, positive, direction_type, is_start)
  IF positive THEN
    angle = (dest_yaw >= start_yaw) ? dest_yaw - start_yaw
                                    : full_turn - start_yaw + dest_yaw
  ELSE
    angle = (dest_yaw <= start_yaw) ? dest_yaw - start_yaw
                                    : dest_yaw - start_yaw - full_turn
  # the destination circle of a same-sign pair sweeps the OTHER way round
  IF NOT is_start AND direction_type IS one of (both-forward, both-backward) THEN
    angle = angle + (angle <= 0 ? +full_turn : -full_turn)
```

**Invariants** — the first half normalizes into a half-open turn in the requested rotational
sense. The second half is the subtle part: on the *destination* circle of a trajectory whose
two legs run the same way, the tangent line arrives on the far side of the circle, so the arc
that actually joins it is the complement. Omitting that correction produces trajectories that
loop almost all the way around the destination circle before arriving — visibly wrong, and
the classic bug in a hand-rolled implementation of this geometry.

## `compute_circles`

**Contract** — given a point with a heading, a linear velocity and an angular velocity,
produces the **two** turning circles tangent to the heading at that point, one to each side.
Fails when the angular velocity is zero, since a body that cannot turn has no turning circle.

```text
FUNCTION compute_circles(point) -> (left_circle, right_circle)
  radius = |linear_velocity| / angular_velocity
  # centres sit one radius to each side, perpendicular to the heading
  right.center = point.position + perpendicular(point.direction) * radius
  left.center  = point.position - perpendicular(point.direction) * radius
```

**Invariants** — the radius is derived from the two velocities and is therefore a property
of *how the creature is moving*, not of the creature. Walking and running have different
turning radii, and the builder tries every enabled velocity precisely to find one whose
radius fits through the gap.

**Invariants** — the heading must already be a unit vector. Nothing normalizes it here; the
build's initialization does.

## `reinit` · construction · destruction

**Contract** — construction binds the optional restrictor-aware object wrapper (which the
path must respect) and marks the destination vertex unknown. `reinit` returns every field to
the reset state described under **State**.
