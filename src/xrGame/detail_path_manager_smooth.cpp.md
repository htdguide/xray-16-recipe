# src/xrGame/detail_path_manager_smooth.cpp

> The smoothing algorithm: reduce a list of navigation cells to a handful of corners, pull each corner outward to widen its turn, and join the corners with circle-line-circle trajectories that respect the body's turning radius.

**Needs** — [`detail_path_manager.h`](detail_path_manager.h.md) · [`ai_space.h`](ai_space.h.md) · [`xrAICore/Navigation/level_graph.h`](../xrAICore/Navigation/level_graph.h.md) · [`restricted_object.h`](restricted_object.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: plane geometry with a per-sample navigation-mesh walk, on a worker thread

## Purpose

This is the file that makes creatures in this game move like bodies rather than like
counters on a grid. Its input is the coarse pathfinder's list of navigation vertices — a
staircase through the mesh. Its output is a dense list of points, each with a position, a
navigation vertex and a velocity identifier, forming a curve that a body with a finite
turning radius can follow without leaving the walkable surface.

The algorithm has four stages, and they are separable:

1. **Initialize** — project everything into the plane, snap both endpoints to genuinely
   interior positions, and build the list of velocities that are allowed on this path.
2. **Find the corners** — reduce the vertex list to the minimum set of points at which the
   creature must actually turn, by repeatedly asking "how far along can I go in a straight
   line".
3. **Improve the corners** — slide each corner away from the obstacle it is hugging, so the
   turn through it is wider and a larger turning radius fits.
4. **Join the corners** — between each consecutive pair, search over velocities and turning
   circles for the fastest trajectory that stays on the mesh.

The geometry in stage four is the classical shortest-path-for-a-bounded-curvature-vehicle
construction: leave the start on one of two turning circles, run along a common tangent,
arrive on one of two turning circles at the destination. Four circle pairs, up to four
tangent solutions, sorted by traversal time.

## State

All working state belongs to the owning [`CDetailPathManager`](detail_path_manager.h.md).
This file adds one scratch list — the temporary path a candidate trajectory is built into
before it is accepted — and the key-point list.

**Invariant** — every stage works in the plane. The vertical coordinate is assigned once, to
the whole finished path, by sampling the navigation mesh. Nothing before that is allowed to
depend on height.

**Invariant** — the navigation vertex of each generated point is obtained by **stepping**
from the previous point's vertex toward the new position, never by looking the position up
in the mesh. A step that leaves the mesh returns an invalid vertex, and that is the primary
way a candidate trajectory is rejected. This is what makes the path *provably* walkable
rather than merely geometrically plausible.

## `build_smooth_path` — the whole pass

**Contract** — builds the detail path from the coarse vertex list. Marks the build failed on
entry and clears that only on success, so every early return is a failure. Runs inside a
profiler zone; typically runs on a worker thread through
[`CDetailPathBuilder`](detail_path_builder.h.md).

```text
FUNCTION build_smooth_path(level_path, intermediate_index)
  failed = true
  invalidate the cached distance-to-target
  IF NOT init_build(...) THEN RETURN          # no usable velocity, or a bad endpoint
  dest_vertex_id = last vertex of level_path

  IF the object has restrictors THEN
    add a temporary border around the start and destination vertices

  finish_params = use_dest_orientation ? the full velocity list : the pivot-in-place list

  IF NOT fill_key_points(...) THEN            # could not reduce to corners
    remove the border; RETURN
  postprocess_key_points(...)
  build_path_via_key_points(...)              # clears `failed` on success
  remove the border
```

**Invariants** — the restrictor border is installed for the duration of the build and
removed on *every* exit path. The border makes the start and destination vertices reachable
even when they sit on the edge of a permitted region, which they routinely do — a creature
standing at the boundary of its restrictor must still be able to build a path out of it.
Leaking the border would leave the restrictor permanently widened.

**Invariants** — which parameter set the final leg uses is decided here. When the caller
demands a specific arrival heading, the final leg may use any of the creature's velocities;
when it does not, the final leg uses the single synthetic "turn in place" entry built in
initialization. This is what makes an arrival without a required heading cheap.

## `init_build`

**Contract** — prepares the plane-space endpoints and the velocity lists. Fails when no
velocity survives the masks, which is the only way this stage can fail.

```text
FUNCTION init_build(level_path, ...) -> (start, dest, straight_index, straight_index_neg)
  current_travel_point = 0; path = empty
  start = (plane(start_position), plane(start_direction), first vertex of level_path)
  validate_vertex_position(start)
  dest  = (plane(dest_position), heading, last vertex of level_path)
     # the heading is the requested one only if an arrival orientation was demanded;
     # otherwise a fixed axis, which the pivot-in-place entry makes irrelevant
  validate_vertex_position(dest)
  corrected_dest = dest.position, with the height taken from the navigation vertex's plane
  normalize both headings; substitute the fixed axis for a degenerate one

  start_params = empty
  FOR EACH (id, params) IN the creature's velocity table
    IF id is not permitted by the velocity mask THEN CONTINUE
    IF id is permitted by the DESIRABLE mask THEN
      push to the FRONT of start_params                 # tried first
      track the fastest forward id  -> straight_index
      track the fastest reverse id  -> straight_index_neg
    ELSE
      push to the BACK                                  # tried only as a fallback
  FAIL IF start_params is empty

  dest_params = [ one synthetic entry: zero linear speed, a full turn of angular
                  speed, an invalid identifier ]        # "pivot on the spot"
```

**Invariants** — the two masks do different jobs and both are needed. The **velocity mask**
is permission: a velocity not in it is unusable. The **desirable mask** is preference: a
velocity in it is tried before the others, by being pushed to the front of the list. Since
the search returns the first success unless minimum-time is requested, preference usually
decides the answer outright.

**Invariants** — the straight-line velocity is the *fastest* permitted-and-desirable one in
each direction, tracked separately for forwards and backwards. The straight leg of every
trajectory uses it, which encodes the policy: turn at whatever speed the corner allows, but
cover the straight stretches as fast as you are allowed to.

**Invariants** — the destination parameter set is a single synthetic entry with **zero**
linear velocity and a full turn of angular velocity, under an invalid identifier. A zero
linear velocity gives a zero turning radius, so the destination "circle" degenerates to the
point itself and the arrival leg is a pivot rather than an arc. The invalid identifier is
never written into a path point because the degenerate arc emits none.

## `validate_vertex_position`

**Contract** — nudges a point that is on or fractionally outside its navigation vertex to a
point definitely inside it. Leaves a point that is already interior alone.

```text
FUNCTION validate_vertex_position(point)
  IF the position is representable AND lies inside point.vertex_id THEN RETURN
  contour  = the polygon of point.vertex_id
  nearest  = the closest point on that contour to the position
  inward   = normalize(centre of the contour's diagonal - nearest)
  point.position = nearest + inward * a small epsilon
```

**Invariants** — after this runs, the point is inside its vertex. Everything downstream —
the straight-line walk, the tangent tests, the arc stepping — assumes a point and its vertex
agree, and a caller's position frequently does not: the creature's own position drifts
fractionally outside its cell as it walks, and a destination handed in by script is
arbitrary.

**Notes** — the inward direction is taken toward the midpoint of one diagonal of the cell,
which is the cell's centre for the square cells the mesh actually uses. The epsilon is the
math layer's large epsilon, chosen to be bigger than the containment test's own tolerance.

## `fill_key_points` — reducing the vertex list to corners

**Contract** — produces the ordered list of points where the path must change direction.
Fails when no progress can be made, which happens when the straight-line walk cannot even
reach the next vertex.

The method is a **binary search for the furthest visible vertex**, repeated:

```text
FUNCTION fill_key_points(level_path, start, dest)
  current = start
  furthest_reached = 0 ; n = last index of level_path
  low = 0 ; probe = n ; high = n
  LOOP
    IF the straight walk from `current` to level_path[probe] leaves the mesh THEN
      high = probe ; probe = midpoint(low, probe)        # too far; come back
    ELSE IF probe is the last index AND the straight walk from `current` to the
            real destination leaves the mesh THEN
      high = probe ; probe = midpoint(low, probe)        # the vertex is visible but
                                                         #   the destination inside it is not
    ELSE
      low = probe ; probe = midpoint(low, high)          # reachable; try further

    IF low and high have converged THEN
      FAIL IF low did not advance past furthest_reached   # no progress: give up
      furthest_reached = low
      emit `current` as a key point
      IF low is the last index THEN
        FAIL IF the straight walk to the destination leaves the mesh
        emit the destination as the final key point ; DONE
      current = level_path[low], at that vertex's own centre
      reset the search window to the whole remaining path
```

**Invariants** — the search is over a *monotone* predicate only by assumption: it presumes
that if a distant vertex is reachable in a straight line then nearer ones are too. That is
false in general — a corridor can bend back into view — and the consequence is a key point
occasionally further back than necessary, never an invalid one. The alternative, a linear
scan, costs the number of vertices per corner instead of its logarithm, and paths routinely
run to hundreds of vertices.

**Invariants** — the no-progress check is the termination guarantee. Without it a
configuration where the walk from a vertex cannot reach even the next vertex loops forever,
which is reachable on a mesh with a one-cell-wide diagonal.

**Invariants** — the last vertex gets an extra test, against the *actual destination* rather
than the vertex centre. The destination is an arbitrary point inside its cell and can be
occluded when the cell's centre is not.

**Notes** — each new anchor is placed at its navigation vertex's own centre rather than at
the position the walk actually reached. Corners therefore sit on cell centres, which is what
makes the improvement stage below both necessary and possible.

## `postprocess_key_points` — widening the corners

**Contract** — replaces each interior corner with a better one. A path of fewer than three
key points is left alone; there are no interior corners to improve.

Because the previous stage anchors corners at cell centres, a path around an obstacle hugs
it at a sharp angle. A sharp corner needs a small turning radius, a small radius needs a low
speed, and a creature that has to slow to a walk at every corner looks wrong. So each corner
is slid along the bisector away from the obstacle until the angle at it is as wide as the
mesh allows.

```text
FUNCTION postprocess_key_points()
  IF fewer than 3 key points THEN RETURN
  IF the last two coincide THEN drop the last
  FOR EACH interior index i
    candidate_a = compute_better_key_point(prev, current, next, forward)
    candidate_b = compute_better_key_point(next, current, prev, reverse)
    key_points[i] = the candidate with the WIDER angle at it
```

**Notes** — the file contains four blocks that compute a vertex identifier, test it, and do
nothing with the result — a self-assignment where a breakpoint used to be. They are dead and
a maintainer's comment in the source says as much. A rebuild deletes them.

## `compute_better_key_point`

**Contract** — slides one corner along the bisector-ish direction away from the corner, as
far as the mesh permits, by bisection. Returns the original corner unchanged when no
improvement is reachable. Pure with respect to the path; queries the navigation mesh.

```text
FUNCTION compute_better_key_point(point0, point1, point2, reversed) -> TravelPoint
  # the direction to slide: directly away from point2, along the point2→point1 line
  slide_dir = normalize(point1.position - point2.position)
  cos_alpha = clamped dot of (that direction) and (point0 - point2)
  # the full slide that would put the corner on the shortcut between point0 and point2
  full  = distance(point1,point2) - distance(point0,point2) / cos_alpha / 2

  low = 0 ; high = 1 ; factor = 1 ; result = point1
  REPEAT
    candidate = point1.position - slide_dir * full * factor
    reachable = candidate is representable
                AND a straight walk from point0 reaches it        (or from point2 if reversed)
                AND a straight walk from it reaches point2        (or point0 if reversed)
    IF reachable THEN low = factor ; result = candidate and its vertex
    ELSE high = factor
    factor = midpoint(low, high)
  UNTIL low and high agree within one percent
  RETURN result
```

**Invariants** — a candidate is accepted only when **both** legs through it stay on the
mesh, checked in the given traversal order. The order matters because the mesh walk is not
symmetric at cell boundaries, which is exactly why the caller computes the point twice, once
in each direction, and keeps the better.

**Invariants** — the bisection converges on the *largest admissible* slide rather than the
first admissible one, because failure moves the upper bound and success moves the lower.

**Notes** — the one-percent convergence tolerance bounds the work at about seven mesh-walk
pairs per corner per direction. That is the tuning knob if this stage ever costs too much.

## `better_key_point`

**Contract** — of two candidate corners, says which gives the wider turn: the one whose
cosine of the angle between the two legs is *smaller*, since a wider turn means the legs point
more nearly opposite. Pure.

## `build_path_via_key_points` — joining the corners

**Contract** — walks the key points in order, computing a trajectory between each
consecutive pair and appending it. On any failure the whole path is discarded and the build
stays failed — there is no partial path.

```text
FUNCTION build_path_via_key_points(start, dest, finish_params, ...)
  s = start
  FOR EACH key point k after the first
    last = k is the final key point
    IF last THEN d = dest
    ELSE      d = k, with its heading set toward the NEXT key point
    IF NOT compute_path(s, d, into the path, start_params,
                        last ? finish_params : start_params, ...) THEN
      clear the path ; RETURN
    IF last THEN BREAK

    # the next leg must start at the heading the previous leg actually ended on
    s = d
    s.direction = the direction of the path's final segment
    WHILE that segment is degenerate: drop the last point and try the one before
    drop the last point                      # it is re-emitted by the next leg
    IF the path is non-empty AND its last point's velocity is a reverse one THEN
      s.direction = -s.direction             # a reversing leg's heading is its facing,
                                             #   not its travel direction
  IF there were no key points at all THEN
    compute one trajectory straight from start to dest, or fail

  add_patrol_point()
  assign the vertical coordinate to every path point from the navigation mesh
  failed = false
```

**Invariants** — each leg's *start heading is read back out of the geometry the previous leg
produced*, never assumed from the key points. A trajectory arrives on an arc, so its exit
heading is tangent to that arc and is not the direction to the next corner. Assuming
otherwise produces a discontinuity at every corner, which the follower turns into a visible
snap.

**Invariants** — the last point of each leg is dropped before the next leg begins, because
the next leg starts by emitting its own start point. Without it the path carries a duplicate
at every corner, and the degenerate segment that creates would make the heading read-back
above fail.

**Invariants** — when the final segment is degenerate, points are popped until a
non-degenerate one is found. A trajectory can end with several coincident samples where an
arc's swept angle rounds to nothing.

**Invariants** — a leg travelled in reverse has its heading negated, because the creature's
*facing* is opposite to its travel. The next leg's turning circles are built from the facing.

**Invariants** — the vertical coordinate is assigned last, to the finished path, in one
pass. Every earlier stage is two-dimensional.

## `compute_path` — searching over velocities

**Contract** — finds a trajectory between two oriented points by trying every combination of
a start velocity and a destination velocity. Returns on the first success, unless
minimum-time is requested, in which case it tries all combinations and keeps the fastest.

```text
FUNCTION compute_path(start, dest, path, start_params, dest_params, straight, straight_neg)
  best = +infinity ; base = current path length
  FOR EACH sp IN start_params                     # desirable velocities come first
    direction_type = both-forward
    apply sp to start
    straight_for_this = straight
    IF sp.linear_velocity is negative THEN
      straight_for_this = straight_neg
      direction_type |= first-leg-reverse
      start.direction = -start.direction
    FOR EACH dp IN dest_params
      apply dp to dest
      IF dp.linear_velocity is negative THEN direction_type |= second-leg-reverse
      IF the first leg is reverse THEN dest.direction = -dest.direction
      IF compute_trajectory(start, dest, scratch, time, sp.id, straight_for_this,
                            dp.id, direction_type) THEN
        IF NOT try_min_time THEN adopt scratch ; RETURN success
        IF time < best THEN best = time ; adopt scratch
  RETURN best is finite
```

**Invariants** — the trajectory's three legs get three *different* velocity identifiers: the
start arc uses the start velocity, the straight uses the fastest permitted one in the
appropriate direction, and the destination arc uses the destination velocity. That is how
one path can record "turn at a walk, sprint the straight, pivot on arrival".

**Invariants** — reversing the start leg negates the heading of *both* points. The whole
trajectory is mirrored, not just its first arc.

**Notes** — the direction-type accumulator is never cleared between destination-parameter
iterations, so a reverse destination velocity leaves its flag set for the subsequent
iterations of the inner loop. With the shipped single-entry destination list the inner loop
runs once and this is unreachable; it becomes a real defect the moment a caller supplies
several destination velocities, which the orientation-demanded path does. Flagged rather
than reproduced.

## `compute_trajectory`

**Contract** — builds both turning circles at each endpoint, enumerates the tangent solutions
between the four circle pairs, discards any whose tangent points are off the mesh, and hands
the survivors to the trajectory builder.

```text
FUNCTION compute_trajectory(start, dest, path, velocities..., direction_type)
  start_circles = compute_circles(start)        # left and right
  dest_circles  = compute_circles(dest)
  candidates = empty
  FOR i IN start_circles, FOR j IN dest_circles
    IF compute_tangent(start, i, dest, j, direction_type) succeeds
       AND both tangent points are representable mesh positions THEN
      append the pair to candidates
  RETURN build_trajectory(start, dest, candidates, path, ...)
```

**Invariants** — at most four candidates, one per circle pair. The tangent points are tested
for mesh validity *before* anything is built, because building a candidate costs an arc walk
and most rejections are cheap.

## `build_trajectory` (candidate selection)

**Contract** — orders the candidates by estimated traversal time and builds them in that
order, returning the first that survives the mesh walk. Restores the path to its prior length
after each failure, so a partial candidate never contaminates the result.

```text
FUNCTION build_trajectory(start, dest, candidates, path, ...)
  FOR EACH candidate
    time = |start arc angle| / start angular speed
         + |dest arc angle|  / dest angular speed
         + tangent length / straight linear speed     # zero if that speed is zero
  sort candidates by time, ascending
  base = current path length
  FOR EACH candidate in that order
    install its circles into start and dest
    IF build_trajectory(start, dest, path, three velocities) THEN RETURN its time
    truncate the path back to `base`
  RETURN failure
```

**Invariants** — the estimate is a genuine traversal time, not a length: arcs are charged at
their angular speed and the straight at its linear speed, so a long fast straight beats a
short slow arc. This is the only place the three velocities are compared against each other.

**Invariants** — a zero straight velocity contributes zero time rather than infinity. That is
the pivot-in-place case, where the tangent length is itself zero.

## `build_trajectory` (three legs) · `build_circle_trajectory` · `build_line_trajectory`

**Contract** — a trajectory is arc, line, arc, built in that order. The first arc reports the
navigation vertex it ended on, which the line then starts its mesh walk from; the second arc
is built without a vertex output. Any leg failing fails the trajectory.

`build_line_trajectory` walks the navigation mesh in a straight line between two plane
points, emitting a sample per cell crossing, or emits a single point when the destination is
inside the current cell. Given no output list it merely tests reachability, which is what the
tangent pre-check uses.

`build_circle_trajectory` is the interesting one:

```text
FUNCTION build_circle_trajectory(point, path, vertex_out, velocity)
  min_dist = 0.1                      # metres: the sampling target
  IF point.radius * |point.angle| <= min_dist THEN
    # the arc is shorter than one sample: emit at most one point and stop
    emit point.position if it is not a duplicate of the path's tail
    RETURN success

  # how many samples: the tighter of two limits
  n = min( |angle| / angular_velocity * 10 ,        # one sample per 100 ms of turning
           radius * |angle| / min_dist )            # one sample per 10 cm of arc
  n = max(n, 1)

  # step around the arc by repeated angle addition, not by recomputing sin and cos
  sin_step, cos_step = sine and cosine of (angle / n)
  FOR i IN 0 .. n (+1 when a vertex is being reported)
    position = centre + radius * (the running angle applied to the start direction)
    vertex = step the navigation mesh from the previous vertex toward position
    FAIL IF that vertex is invalid
    emit (position, vertex, velocity)
    advance the running angle by one step
  IF a vertex was requested THEN report the final one
  ELSE reverse the points just emitted
```

**Invariants** — the sample count is the *minimum* of a time-based and a distance-based
limit, which is the whole sampling policy: a tight slow turn is sampled by time so the
follower gets a command every hundred milliseconds, and a wide fast turn is sampled by
distance so the polyline stays within ten centimetres of the true arc. Either limit alone
produces either thousands of samples on a wide arc or a visibly polygonal tight one.

**Invariants** — the running angle is advanced by **angle addition** — one sine and one
cosine for the whole arc, then repeated application of the step — rather than by a
trigonometric call per sample. This is not merely an optimization: it makes the samples
exactly equally spaced in angle, and the accumulated drift over the few dozen samples of an
arc is below the mesh's tolerance.

**Invariants** — the **destination** arc is emitted in reverse. It is generated from the
destination point backwards along the arc, because the mesh walk must start from a vertex
that is known good, and at that stage only the destination's vertex is. Reversing afterwards
puts it in travel order. The presence or absence of the vertex output is what distinguishes
the two cases, which is an overloading of one parameter that a rebuild should split.

**Notes** — the degenerate-arc branch emits a point only when it is not a duplicate of the
path's tail, or when a vertex is being reported. The duplicate check exists because the leg
that follows will emit its own start point.

## `compute_tangent`

**Contract** — given two circles and the headings at the points they were built from, finds
the tangent line joining them that the body can actually traverse, and the swept angles on
both arcs. Fails when no such tangent exists. This is the geometric core and the densest
function in the file.

```text
FUNCTION compute_tangent(start, start_circle, dest, dest_circle, direction_type)
  start_cross = cross(start.direction, start.position - start_circle.center)
  dest_cross  = cross(dest.direction,  dest.position  - dest_circle.center)
      # the sign of each says which way round its circle the body turns

  IF start_cross and dest_cross have the SAME sign THEN
    # both arcs turn the same way: the joining tangent is an OUTER tangent
    IF the circles are concentric THEN
      IF their radii also match THEN
        # one circle: the "tangent" degenerates to a single arc and a zero-length line
        emit the destination bearing's point, the swept angle, and a zero second angle
        RETURN success
      RETURN failure                 # concentric, different radii: no tangent exists
    alpha = arccos( (start_radius - dest_radius) / centre_distance )
    FAIL IF |start_radius - dest_radius| exceeds the centre distance
  ELSE
    # the arcs turn opposite ways: the tangent CROSSES between them — an inner tangent
    FAIL IF start_radius + dest_radius exceeds the centre distance
    alpha = arccos( (start_radius + dest_radius) / centre_distance )
    the destination bearing is taken half a turn round     # the crossing

  # two mirror-image solutions; take the one whose travel direction agrees with the arcs
  tangent points = the circles' points at (centre bearing ± alpha)
  IF the `+` solution's direction agrees with both arcs THEN use it ELSE use the `-`
  swept angles = assign_angle(...) on each circle, with `is_start` false for the second
```

**Invariants** — the sign of each cross product is the body's rotational sense on that
circle, fixed by its heading. Same signs mean the two arcs turn the same way and the joining
line must not cross between the circles; opposite signs mean it must. Choosing the wrong
family gives a tangent the body would have to teleport onto.

**Invariants** — the cosine argument is **clamped just inside ±1** before the inverse cosine.
The exact tangency case — circles touching, or one enclosing the other exactly — puts the
argument on the boundary, and floating-point rounding there produces an undefined result
rather than a zero angle. The clamp costs a negligible angular error and removes a crash.

**Invariants** — the near-tangency tests admit the *approximately* equal case rather than
only the strictly-less case. Circles that touch to within the epsilon have a valid tangent,
and rejecting them makes a creature refuse a corner it can just barely make.

**Invariants** — the swept angle on the destination circle is computed with the
non-start correction described in
[`detail_path_manager.cpp`](detail_path_manager.cpp.md) under `assign_angle`.

**Notes** — the concentric-equal-radii case is not a curiosity: it arises whenever a leg
starts and ends at the same speed and the two key points are close, which is common on a
gentle path, and it is handled by collapsing the trajectory to one arc.

## `coincide_directions`

**Contract** — a helper for the tangent choice: given the two tangent points and both cross
products, says whether travelling from the start tangent point to the destination tangent
point agrees with the rotational sense of the arcs. Handles a zero start cross product — the
start point exactly on its own circle's centre line — by asking the question from the
destination end instead.

## `add_patrol_point`

**Contract** — records where the path's *real* destination sits, and for a patrol path
extends the path beyond it.

```text
FUNCTION add_patrol_point()
  last_patrol_point = last index of the path
  IF the path has fewer than two points, or this is not a patrol path,
     or the extrapolation length is zero THEN RETURN
  direction = the path's final segment, flattened and normalized
  RETURN IF that segment is degenerate
  walk the mesh straight from the path's end, `extrapolate_length` metres along
    that direction, appending the samples
```

**Invariants** — the extrapolated tail carries the same velocity identifier as the final
point, so the creature keeps its speed through the waypoint. This is the whole reason the
tail exists: a patrolling creature that stopped exactly on each waypoint would decelerate and
accelerate at every one.

**Invariants** — the recorded last-patrol index is what `completed` and `valid` test against
for a patrol path; the tail is deliberately past the destination and must not count as
failure to arrive. See [`detail_path_manager_inline.h`](detail_path_manager_inline.h.md).

**Notes** — the mesh walk that lays the tail down is free to stop early when it runs out of
walkable surface, so the tail is at most the extrapolation length and often shorter. That is
correct and needs no handling: a shorter tail merely means less carry-through.

## `sin_apb` · `cos_apb` · `is_negative`

**Contract** — the angle-addition identities, used to step around an arc without recomputing
trigonometric functions per sample, and a sign test that treats an exact zero as
non-negative. The last matters: a velocity of exactly zero is the pivot-in-place case and must
not be classified as reverse.
