# src/xrPhysics/tri-colliderknoopc/dTriCylinder.cpp

> Cylinder against triangle — the case that carries every creature in the game, because a
> character is a cylinder standing on a triangle soup.

**Needs** — [`dTriCylinder.h`](dTriCylinder.h.md) · [`dcTriListCollider.h`](dcTriListCollider.h.md) · [`dTriColliderCommon.h`](dTriColliderCommon.h.md) · [`../dcylinder/dCylinder.h`](../dcylinder/dCylinder.h.md) · [`../MathUtils.h`](../MathUtils.h.md) · [`../ExtendedGeom.h`](../ExtendedGeom.h.md) · [`../PHWorld.h`](../PHWorld.h.md) · [`../../xrCDB/xr_area.h`](../../xrCDB/xr_area.h.md) · [Seam: Static collision database](../../../SYSTEM-REQUIREMENTS.md#seam-static-collision-database)
**Used by** — [`dTriCylinder.h`](dTriCylinder.h.md)
**Tier floor** — T1: separating-axis arithmetic over raw arrays inside the solver's narrow
phase.

## Purpose

The character controller's collision shape is a cylinder
([`../PHCharacter.h`](../PHCharacter.h.md)), and the level is a triangle soup, so this file is
what the player and every creature actually stand on. It is the longest of the three primitive
cases and the one whose failure modes are most visible: a wrong normal here is a player
sliding down a flat floor, snagging on a seam between two floor tiles, or being launched off a
step.

Two routines, the same split as its siblings: the ordinary case with a full axis search and
engagement bookkeeping, and a recovery case that just pushes out along a plane.

## Stateless.

## `dTriCyl` — the ordinary case

**Contract** — report up to three contacts between an oriented cylinder and a triangle the
cylinder's centre is in front of.

The axis set is thirteen, in four groups:

```text
  code 0      the triangle's plane normal                # the common case: standing
  codes 1..3  the cylinder's own axis, tested against
              each triangle vertex                       # a vertex on a flat end
  codes 4..6  for each vertex: the perpendicular from
              the cylinder's axis to that vertex         # a vertex on the barrel
  codes 7..9  for each triangle edge that the cylinder's
              axis actually crosses: the cross of that
              edge with the axis                         # an edge across the barrel
  codes 10..12 for each edge: the axis through the point
              where the cylinder's RIM meets that edge   # a rim on an edge
```

**Invariants** — several rules govern the search:

*Groups 1–3 and 4–6 only count when all three triangle vertices lie on the same side of the
cylinder along the axis being tested.* A triangle straddling the cylinder is not separated by
that axis.

*Within group 4–6, only the vertex that is deepest of the three is considered*, and only if it
is deeper than any previously accepted vertex axis. A cylinder resting on a corner is pushed
out by the corner it is most on, not by whichever was examined last.

*Groups 7–9 are gated by an explicit crossing test*: the routine first asks whether the
cylinder's finite axis segment actually crosses the triangle's edge within both the edge's
length and the cylinder's half-length. If not, the axis is not a separating candidate at all
and is skipped.

*Group 10–12 is the rim case and requires its own construction* — see below.

*If any axis separates the pair, the answer is immediately "no overlap".*

**The rim-against-edge construction** (codes 10–12) is the one a rebuilder will not invent:

```text
FOR EACH triangle edge e
  offset := the perpendicular from the edge's line to the cylinder's centre
  pick the cylinder END that offset points toward; call its centre C
  intersect the edge's line with the rim circle of radius r about C
      # if the line misses the circle, fall back to its closest point
  at that intersection, take the rim's TANGENT direction
  axis := cross(e, tangent)
  keep it only if it separates the triangle's remaining vertex on the same side
```

Only after the axis survives that construction is the usual projection test applied, using the
cylinder's full reach formula ([`dTriCylinder.h`](dTriCylinder.h.md)).

**Notes on the crossing test** — deciding whether the cylinder's axis crosses a triangle edge
is a closest-approach between two lines, with two bounds: the crossing parameter must be within
the edge's length, and the projection onto the cylinder's axis must be within its half-length.
Both bounds matter. Without the first a distant edge's extended line is treated as an
obstacle; without the second a step below the cylinder's foot pushes it sideways.

**Contact synthesis** by winning code:

- **code 0, the triangle's face** — the cylinder is standing on, or lying against, the
  surface. One contact at the deepest rim point, plus the ±60° rim manifold
  ([`dTriColliderCommon.h`](dTriColliderCommon.h.md)) if the cylinder's axis is within 45° of
  the triangle normal, or one contact at the *other end* of the cylinder if it is not. Every
  one of the extra points is accepted only if it is both still penetrating **and** over the
  triangle's face.
- **codes 1–6, a vertex** — one contact at that vertex. Engagement is consulted first: if
  this triangle's vertex has already been claimed by a neighbour, the whole call returns
  nothing. Otherwise the vertex is claimed. The normal is the cylinder's axis for codes 1–3
  and the constructed perpendicular for codes 4–6.
- **codes 7–12, an edge** — one contact at the crossing point, with the constructed axis as
  the normal. The corresponding *edge* engagement bit is consulted and claimed the same way.

**Invariants** — the extra manifold points in the face case are tested for containment in the
triangle, which the box case does not do. That difference is not an inconsistency: a cylinder's
rim points are spread a full radius from the contact and routinely land off the triangle,
whereas a box's walk-along-the-edge points stay near the original contact. Without the
containment test a creature standing near the edge of a floor tile is pushed by points hanging
over the void.

**Notes** — the engagement checks are *early returns from the whole routine*, not just from
the contact. A cylinder whose vertex contact is refused produces nothing at all this triangle,
rather than falling back to a face contact — which is right, since it was not over the face.
See [`dcTriListCollider.h`](dcTriListCollider.h.md) for why this bookkeeping exists; without
it, a character crossing the seam between two floor triangles is pushed twice and visibly
hitches, which is the most conspicuous artefact a triangle-soup world can produce.

## `dSortedTriCyl` — the recovery case

**Contract** — the cylinder is *behind* the triangle's plane. The normal is the triangle's,
negated; no axis search, no containment, no engagement.

```text
FUNCTION cylinder_vs_plane(tri_normal, triangle, distance, cylinder, max_contacts) -> int
  IF distance > 0 THEN RETURN 0             # not actually behind it
  depth := cylinder_reach(tri_normal) - distance
  IF depth < 0 THEN RETURN 0

  first contact := the deepest rim point on the facing end
  IF the cylinder's axis is within 45° of the normal
    add the two ±60° rim points, each if its own depth is positive
  ELSE
    add one point at the other end, if its depth is positive
  cap at three; every contact carries tri_normal and the triangle's material
```

**Invariants** — the extra points' depths are computed from the depth **at the disc's centre**
plus the projection of their rim offset onto the normal, not recomputed from scratch. The
centre depth is derived once by subtracting the deepest point's rim offset from the overall
depth, and the three points then differ only by their offsets. That is both cheaper and, more
importantly, *consistent*: three points whose depths were computed independently can disagree
by rounding and give the solver a slightly tilted manifold.

**Notes** — the 45° switch between the three-point rim manifold and the two-end manifold is
the same decision as in [`../dcylinder/dCylinder.cpp`](../dcylinder/dCylinder.cpp.md) and in
the ordinary case above, and it means the same thing: a round face lying flat needs three
points to stop it tipping in two directions, a cylinder lying on its side needs one per end to
stop it rotating about the contact line.

Unlike the ordinary case, the recovery case does *not* test whether its contact points lie
over the triangle. A shape being extracted from inside the geometry is not over anything, and
demanding containment there would leave it stuck.
