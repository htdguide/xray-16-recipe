# src/xrGame/min_obb.cpp

> Fits the tightest oriented box around a set of points, by searching orientation space for the rotation that minimizes the enclosed volume.

**Needs** — [`magic_box3.h`](magic_box3.h.md) · [`magic_minimize_nd.h`](magic_minimize_nd.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: numeric optimization over a point set

## Purpose

An axis-aligned box around a diagonal cluster of points is mostly empty, and empty volume is
wasted collision tests. This file finds the box that is actually tight: it parameterizes
orientation with three angles, defines volume as a function of those angles, and hands that
function to the general minimizer. The result is used wherever a fitted volume must be
compact — bone hit regions, generated collision shapes.

The method is *not* exact. The true minimum-volume box is found by a combinatorial
construction over the convex hull; this is a numeric search that gets close and is far
simpler. That trade is the file's central decision.

## State

`Stateless.` — the point set is passed through the minimizer as an opaque value and never
copied.

## `MagicMinBox`

**Contract** — takes a point count and a point array, returns the fitted oriented box. Does
not allocate. Cost is roughly (number of sampled orientations) × (number of points), which
makes it a load-time or tool-time operation, never a per-frame one.

Two phases: a coarse grid scan to choose a starting orientation, then a local search from
there.

```text
FUNCTION min_box(points) -> Box
  domain = angle0 in [0, pi], angle1 in [0, pi/2], angle2 in [0, pi]

  # phase one: sample a 4 x 4 x 4 grid over the domain, keep the best
  best = the grid point with the smallest enclosing volume

  # phase two: local search from there, over the same domain
  angles = minimize(volume, domain, starting at best)

  RETURN the enclosing box at those angles
```

**Invariants** — the grid scan exists because the volume function has many local minima and
the local search finds only the one it starts in. Four samples per axis — sixty-four
orientations — is the shipped value and has no derivation beyond being enough to land in the
right basin on real point sets while staying cheap. It is also *inclusive* of both endpoints,
so the spacing is a third of each range, not a quarter.

The angle domain is half of what a naive parameterization would use, and this is the one
genuinely clever part. The first two angles are spherical coordinates of a rotation *axis*
and the third is the rotation about it. A box is symmetric under a half turn about any of its
own axes, so orientations related by that symmetry produce the same box; restricting the
polar angle to a quarter turn and the other two to a half turn covers every distinct box
exactly once. Searching the full sphere would triple the work and find the same answer three
times.

## `Volume` — the objective

**Contract** — given three angles and the point set, returns the volume of the axis-aligned
box that encloses the points *after* rotating them by those angles. Pure. This is the
function the minimizer is searching.

```text
FUNCTION volume(angles, points) -> real
  axis = spherical direction from angles[0] (azimuth) and angles[1] (polar)
  rot  = rotation of angles[2] about that axis
  transform the first point; seed both bounds with it
  FOR EACH remaining point
    transform it; widen the bounds
  RETURN the product of the three bound spans
```

**Notes** — the bound update per axis is written as an if/else-if pair: a coordinate below
the current minimum is *not* also compared against the maximum. This is correct, because a
value cannot be below the running minimum and above the running maximum at the same time —
except on the very first comparison, which is why both bounds are seeded from a real point
rather than from infinities. A rebuild that seeds with infinities must use independent
comparisons.

Every evaluation rebuilds the rotation from scratch and re-transforms every point. The
minimizer evaluates this hundreds of times, so this loop is the whole cost of the fit. A
rebuild can hoist nothing — each evaluation is a different rotation — but it can transform
points in bulk.

## `MinimalBoxForAngles`

**Contract** — builds the actual box at a given orientation. Repeats the volume function's
bound computation and then converts the result back to world space: the box's centre is the
midpoint of the rotated bounds *rotated back*, its axes are the rotation's columns, and its
half-extents are half the bound spans.

**Invariant** — the axes are taken as the rotation matrix's **columns**, not its rows. The
points were transformed *by* the rotation to compute the bounds, so the box's axes in world
space are the inverse rotation's rows, which for a rotation are the original's columns. A
rebuild that takes rows produces a box that is the mirror of the right one and encloses
nothing.

**Notes** — this function duplicates the objective's bound loop exactly. The duplication is
real and avoidable: one function returning the bounds, with the objective taking the volume
and this one taking the box, is the same code once.

## `FromAxisAngle` · `GetColumn`

**Contract** — build a rotation matrix from an axis and an angle; read one column of a
matrix as a vector. Both are local helpers that exist because the engine's own matrix type
did not offer them in the form this file wanted. They encode no decision; a rebuild uses
whatever its own rotation representation provides.

**Invariant** — the axis must be unit length. It is, because it is constructed from sines and
cosines of two angles, but nothing checks.
