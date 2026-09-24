# src/xrGame/trajectories.cpp

> Does this thrown or jumping thing clear the geometry between here and there? Answered by
> approximating a parabola with as few straight segments as its curvature allows.

**Needs** — [`trajectories.h`](trajectories.h.md) · [`Level.h`](Level.h.md) · [`xrPhysics/IPHWorld.h`](../xrPhysics/IPHWorld.h.md) · [Seam: Static collision database](../../SYSTEM-REQUIREMENTS.md#seam-static-collision-database)
**Used by** — [`trajectories.h`](trajectories.h.md)
**Tier floor** — T2: iterative geometry against the collision database.

## Purpose

Two questions in the game have the same shape: *can this creature jump from here to there
without hitting anything*, and *can this stalker throw a grenade at that spot without it
bouncing off a doorframe*. Both are a ballistic arc tested against the level's collision
geometry, and both are answered here.

A parabola cannot be tested directly — the collision database answers about straight segments
and boxes. The question is therefore how finely to chop the arc. Chopping uniformly is
wasteful where the arc is nearly straight and inaccurate where it bends. This file chops
**adaptively**: it repeatedly asks *how far ahead can I go before a straight line departs
from the true arc by more than a tolerance*, tests that segment, and continues from its end.

The tolerance is the module's whole tuning: it is one tenth of a world unit, which is the
scale at which clipping a doorframe starts to be visible.

## State

Stateless. Callers supply scratch space for the collision query results, so that a routine
called per creature per decision does not allocate.

```text
RECORD TrajectoryPick        # only produced for the diagnostic overlay
  center, x_axis, y_axis, z_axis, sizes : vector
  invert_x, invert_y, invert_z          : bool
```

## The arc

```text
FUNCTION position_at(start, velocity, gravity, t) -> vector
  RETURN start + velocity * t + gravity * t * t / 2
```

Gravity is taken from the physics world rather than being a constant here, so a level with
altered gravity produces arcs the thrower and the physics agree on.

## Segment error, and where it is largest

```text
FUNCTION segment_error(t_low, t_high, start, velocity, gravity) -> real
  t_mid  := (t_low + t_high) / 2          # the point of maximum departure
  a := position_at(t_low);  b := position_at(t_high);  m := position_at(t_mid)
  RETURN distance from m to the straight line a-b
```

**Invariants** — the maximum departure of a parabola from the chord between two of its points
is always at the **midpoint in time**, and this is exact, not an approximation: the difference
between the arc and the chord is itself a parabola in time with its extremum at the mid-time.
That fact is what makes the whole adaptive scheme cheap — finding the worst point costs one
evaluation rather than a search. A rebuild must not replace it with a sampled maximum.

Air resistance would break the identity; the projectile code elsewhere that does model it
carries its own note that the property still holds for its particular formulation.

## Choosing a segment length

**Contract** — given a start time and an upper bound, find the latest end time whose chord
stays within the tolerance. Binary search, terminated on a time resolution corresponding to a
tenth of a world unit of travel.

```text
FUNCTION select_segment_end(t_start, t_max, start, velocity, gravity, tolerance) -> real
  low := t_start;  high := t_max;  probe := t_max
  time_resolution := 0.1 / speed          # a tenth of a unit of travel, expressed as time
  WHILE NOT close(low, high, time_resolution)
    IF segment_error(t_start, probe, ...) < tolerance
      low := probe                        # this reach is acceptable; try further
    ELSE
      high := probe
    probe := (low + high) / 2
  RETURN low                              # always an acceptable reach
```

**Invariants** — the search always returns a value that *passed*, never one that failed, so
the segment handed to the collision test is always within tolerance. The termination
threshold is expressed in **time** but derived from distance, so the search stops when
further refinement would move the endpoint less than a tenth of a unit — the same scale as
the tolerance itself, which is what stops it refining forever on a nearly-straight arc.

## Testing one segment

**Contract** — tests the chord between two times against the level geometry, in one of two
modes. With a zero box size it is a ray; with a non-zero one it is an oriented box swept along
the chord, which is how a grenade's or a creature's *volume* is accounted for rather than
just its centre line. Temporarily disables the thrower and an optionally named ignored object
so the query cannot hit them, and restores their state afterwards. Writes out the collision
position in ray mode.

**Invariants** — the return convention is inverted and a rebuild must not copy the spelling:
the function reports **true when the segment is clear**. Its ray branch says so by comparing
the returned range against the full segment length; its box branch says so by negating the
box query's hit answer. The two branches also differ in what they produce — only the ray
branch fills in a collision position — which is why the caller's termination test consults
that position only in the mode that supplies one.

Segments shorter than a hundredth of a unit are reported clear without being tested, because
a degenerate segment has no direction to trace along.

**Notes** — two defects are present and reproducible. In the box branch, the box's lateral
axis is computed into a variable that shadows the one the caller reads, so for a trajectory
with any horizontal component the box's orientation is built from an unwritten axis; the
other branch, for a purely vertical trajectory, is correct. And the diagnostic pick records
are appended without bound in builds where the overlay is compiled in, cleared only by the
top-level entry point. A rebuild should fix the first and must not reproduce it; the second
is diagnostic only.

## `trajectory_intersects_geometry`

**Contract** — the entry point. Walks the arc from time zero to the given flight time in
adaptive segments and reports whether any of them hits geometry. Returns **true when the
trajectory is obstructed**. Does not allocate; uses the caller's scratch. Blocks only on the
collision database.

```text
FUNCTION trajectory_intersects_geometry(flight_time, start, end, velocity, ...) -> bool
  gravity   := down * physics_world.gravity
  tolerance := 0.1                          # world units
  low := 0
  LOOP
    t := select_segment_end(low, flight_time, start, velocity, gravity, tolerance)
    IF NOT segment_is_clear(low, t, ...)
      # a hit — unless we are at the very end and landed where we meant to
      IF close(t, flight_time) AND collision_position IS within 0.2 of end
        BREAK                               # that "hit" is the intended landing
      RETURN true
    IF close(t, flight_time)
      BREAK                                 # walked the whole arc with nothing in the way
    low := t
  RETURN false
```

**Invariants** — the landing exemption is the decision that makes the routine usable. Every
successful throw ends by hitting something — the floor at the target. Without the exemption
every legal grenade throw would be reported as obstructed. The test is: this is the last
segment, *and* the thing we hit is within a fifth of a unit of where we were aiming. Anything
further away is a real obstruction.

**Notes** — the segment-length search is called with the trajectory's **start position in
place of its velocity**, so the segment lengths chosen are those of a different arc than the
one being tested. The consequence is not a wrong answer but a wrong segmentation: the
collision tests still walk the true arc from time zero to the flight time, in segments whose
lengths were chosen against nonsense. Depending on the numbers this makes the walk finer than
needed (slow) or coarser (a thin obstacle can be chorded over). A rebuild must pass the
velocity, and should expect the tuning of the tolerance to need revisiting once it does,
because the shipped behaviour was tuned with this in place.

## What could not be recovered

- The tenth-of-a-unit chord tolerance and the fifth-of-a-unit landing tolerance are bare
  numbers; the first is plausibly the visible-clipping scale but nothing says so.
- Whether the velocity-for-position argument, the shadowed box axis and the unbounded
  diagnostic accumulation were known. None is commented.
