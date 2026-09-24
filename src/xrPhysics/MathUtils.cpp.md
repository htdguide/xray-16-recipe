# src/xrPhysics/MathUtils.cpp

> An unused exact ray-versus-capped-cylinder intersection and the benchmark that
> was written to justify it.

**Needs** — [`MathUtils.h`](MathUtils.h.md) · [`utils/xrMiscMath`](../utils/xrMiscMath/README.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: closed-form geometry, no device or layout concern.

## Purpose

This file is **dead code in the shipped engine**, and saying so is more useful than
describing it as if it ran. Nothing calls either of the two things it defines. A rebuilder
should read it, decide whether the capsule test is worth keeping, and otherwise skip the
file entirely.

Its interest is historical and diagnostic: it records an attempt to replace the shared math
layer's cylinder intersection with a faster one, together with the measurement that was
supposed to settle the question. The measurement's result is not recorded anywhere, and the
replacement was never adopted — which is itself the answer.

## `RAYvsCYLINDER`

**Contract** — exact intersection of a ray with a *capped* cylinder (a cylinder closed by
two hemispherical caps, i.e. a capsule): given the cylinder's centre, axis, half-height and
radius, a ray origin and direction, and an in/out maximum distance, report whether the ray
strikes within that distance and shrink the distance to the hit. A culling flag decides
whether a hit behind the origin — that is, a ray starting inside — counts.

**Invariants** — the distance parameter is both the search budget and the result, so a
caller testing many shapes gets nearest-hit semantics for free by reusing it.

The algorithm is a case analysis rather than a single formula, which is where its claimed
speed comes from — most rays are rejected before any square root:

```text
FUNCTION ray_vs_capsule(centre, axis, half_height, radius, origin, dir, io_dist, cull) -> bool
  v   = centre - origin
  cos = dot(dir, axis)            # ray/axis alignment
  # three regimes, chosen so the general case never divides by a near-zero sine
  IF  1 - cos^2  is negligible    RETURN parallel_case()     # ray runs along the axis
  IF      cos^2  is negligible    RETURN perpendicular_case()
  # general case: find the closest approach of the two lines
  t_ray  = (dot(v,dir) - cos*dot(v,axis)) / (1 - cos^2)
  t_axis = (cos*dot(v,dir) - dot(v,axis)) / (1 - cos^2)
  IF squared distance at closest approach > radius^2   RETURN false   # the early out
  # the chord the ray cuts through the infinite cylinder, projected onto the axis,
  # says which of five regions the entry and exit points fall in:
  #   both above the top cap, both below the bottom cap, both within the barrel,
  #   or straddling one cap.  Each region has its own closed-form hit.
  RETURN solve_for_region(...)
```

**Notes** — the early rejection on closest-approach distance is the whole design. The five
region cases exist because a capsule is three surfaces glued together and a ray may enter
through one and leave through another; getting the *entry* surface right is what the case
analysis buys, and it is where a rebuilder will spend their debugging time.

The culling flag's meaning is worth restating: with culling on, a ray whose first
intersection is behind its origin reports *no hit* rather than reporting the exit point.
That is bullet semantics — you cannot shoot something you are already inside.

## the benchmark

**Contract** — runs a fixed set of hand-chosen cases whose expected answers are written as
trailing comments, then a million random ray/capsule pairs through both this routine and the
shared math layer's equivalent, and logs the elapsed time of each.

**Notes** — the hand-chosen cases are the only specification of the routine's edge behaviour
that exists, and they are *commented expectations, not assertions*: nothing checks them, and
the results are discarded. A rebuilder who keeps the capsule test should turn those comments
into real assertions, which is a half-hour of work and is the only test coverage this
chapter has ever had.

The random loop contains a bug that makes it not measure what it claims: the random
cylinders it generates each iteration are never passed to the routine under test, which is
called with the fixed cylinder from the start of the function. Only the rays vary. That is
worth knowing before trusting any timing conclusion drawn from it.
