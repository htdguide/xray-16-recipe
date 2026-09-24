# src/xrGame/MathUtils.cpp

> A hand-written ray-versus-capped-cylinder intersection with its own correctness and speed harness, kept beside the math header but wired to nothing.

**Needs** — [`MathUtils.h`](MathUtils.h.md) · [`utils/xrMiscMath`](../utils/xrMiscMath/README.md)
**Used by** — reached through its declarations in [`MathUtils.h`](MathUtils.h.md); callers name that, not this file.
**Tier floor** — T1: a branch-dense numeric kernel written against a competing implementation's timing

## Purpose

The math header is inline and needs no implementation file. This one holds a single routine
that was written to replace the engine's existing cylinder intersection, together with the
harness used to check it and to time it against the incumbent. Neither is reachable from the
rest of the engine: the routine is file-local and the harness is called from nowhere.

It survives in the recipe because the *decision it records* is useful — someone measured a
hand-rolled capped-cylinder test against the engine's and the replacement did not land — and
because the case analysis below is the part a rebuild would otherwise have to rediscover if
it ever needs this primitive. A rebuild may delete the file outright.

## State

`Stateless.`

## `RAYvsCYLINDER`

**Contract** — intersects a ray with a *capped* cylinder (a cylinder with hemispherical ends,
i.e. a capsule as the physics layer means it). Takes the cylinder, the ray origin and
direction, an in/out distance that is both the search limit and the result, and a culling
flag. Returns whether a hit nearer than the incoming limit was found, and overwrites the
limit with the hit distance when so. With culling on, an intersection behind the origin is
not a hit; with it off, the ray may start inside the volume and the far exit is reported.

**Invariants** — the incoming distance is an upper bound and must be respected: a hit beyond
it is a miss. This is what lets a caller test a list of shapes and keep the nearest without
sorting.

```text
FUNCTION RAYvsCYLINDER(cylinder, origin, direction, in_out_limit, cull) -> bool
  # Work in the cylinder's own frame: everything below is expressed in terms of the
  # projections of the origin-to-centre vector onto the axis and onto the ray.
  to_centre     = cylinder.centre - origin
  cos           = dot(direction, cylinder.axis)
  along_axis    = dot(to_centre, cylinder.axis)
  along_ray     = dot(to_centre, direction)
  half_height   = cylinder.height / 2
  radius_sq     = cylinder.radius^2

  IF the ray is PARALLEL to the axis THEN
    # A one-dimensional problem: the ray either misses the circular cross-section
    # entirely, or enters at (projection - (chord half-length + half height)).
    ... near/far selection under the cull flag ...
  ELSE IF the ray is PERPENDICULAR to the axis THEN
    # Two sub-cases: the ray passes beside the barrel, or beside one of the caps, and
    # the cap case degenerates to a sphere test.
    ...
  ELSE
    # General case. Find the closest approach of the two lines (ray and axis), which
    # gives a distance along each. If that separation exceeds the radius there is no
    # intersection with the INFINITE cylinder and therefore none with the capped one.
    nearest_sq = separation of the two lines at closest approach
    IF nearest_sq > radius_sq THEN RETURN false

    # The chord the ray cuts through the infinite cylinder, expressed as the interval
    # [entry, exit] measured ALONG THE AXIS. Comparing that interval against the caps
    # decides which surfaces the ray actually touches.
    chord_sq = radius_sq - nearest_sq
    half     = sqrt(chord_sq * cos^2 / sin^2)
    entry    = axis_at_closest - half
    exit     = axis_at_closest + half

    IF entry >  half_height THEN   solve against the upper cap sphere only
    IF exit  < -half_height THEN   solve against the lower cap sphere only
    OTHERWISE the interval overlaps the barrel, and one of four sub-cases applies
      depending on the sign of cos and on whether each end of the interval falls
      inside the caps:
        both ends inside   -> barrel only
        upper end outside  -> barrel entering, upper sphere exiting
        lower end outside  -> lower sphere entering, barrel exiting
        both ends outside  -> lower sphere entering, upper sphere exiting
  END IF

  # Every branch ends the same way:
  #   take the near root; if it exceeds the limit, miss.
  #   if it is negative the origin is inside: miss when culling, otherwise take the
  #   far root, and miss if that is negative too.
  #   otherwise record it in the limit and hit.
```

**Notes** — the whole structure is the answer to one question: *which of the three surfaces
does the ray hit first* — the barrel, the upper cap or the lower cap. Rather than intersect
all three and take the nearest, the routine computes the chord interval once and uses its
position relative to the caps to pick the right surface directly. That is the optimization,
and it is why the case analysis is so wide.

The sign of the axis-direction dot product selects between mirrored sub-cases, doubling the
branch count without changing the mathematics. A rebuild can fold those by canonicalizing the
axis to point along the ray first, at the cost of one conditional negation.

Both degenerate branches (exactly parallel, exactly perpendicular) are separate code rather
than falling out of the general case, because the general case divides by the squared sine of
the angle between ray and axis, which is zero for a parallel ray.

## `capped_cylinder_ray_collision_test`

**Contract** — a correctness and performance harness. It runs a fixed set of rays against a
unit cylinder, with the expected answers written beside each call as comments, then times a
million random cases of this routine against a million of the engine's own cylinder
intersection and logs both.

**Notes** — the expected results are comments, not assertions; nothing fails if the routine
regresses. The harness had to be read by a person watching a debugger. A rebuild that keeps
this primitive should turn those comments into real assertions — they are a usable test
oracle as they stand, covering the inside, outside, parallel, perpendicular and oblique
cases, and the culling flag in each.

That a benchmark exists at all, and that the routine it favours was never wired in, is the
file's real content: the replacement was measured and not adopted.
