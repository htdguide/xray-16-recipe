# src/xrPhysics/dcylinder/dCylinder.cpp

> The cylinder primitive the dynamics library lacks, and its collision against boxes, spheres,
> other cylinders, planes and rays.

**Needs** — [`dCylinder.h`](dCylinder.h.md) · [`../tri-colliderknoopc/dTriColliderCommon.h`](../tri-colliderknoopc/dTriColliderCommon.h.md) · [`../tri-colliderknoopc/dTriCylinder.h`](../tri-colliderknoopc/dTriCylinder.h.md) · [`../ode_include.h`](../ode_include.h.md) · [Seam: Rigid-body dynamics](../../../SYSTEM-REQUIREMENTS.md#seam-rigid-body-dynamics)
**Used by** — [`dCylinder.h`](dCylinder.h.md)
**Tier floor** — T1: a user-registered shape kind, with hand-rolled separating-axis tests over
raw coordinate arrays inside the solver's narrow phase.

## Purpose

The engine's characters, its vehicles' wheels and several of its props are cylinders, and the
shipped dynamics library offers only spheres, boxes and capsules. A capsule is not an
acceptable substitute: a capsule cannot stand on its rim, cannot rest flat on its end, and
rolls where a cylinder should sit. So the cylinder is added as a user shape kind, complete
with its own narrow phase against every primitive it must meet.

This is the largest single algorithm in the chapter and it is all one technique —
**separating-axis testing followed by contact-manifold synthesis** — applied five times. The
structure repeats, so read the box case carefully and the rest follows.

## State

```text
RECORD Cylinder
  radius : real
  length : real        # full length, along the shape's LOCAL Y axis
  # invariant: both strictly positive; enforced on create and on every set
```

**Invariants** — the axis is local **Y**. The field is named for Z in the original, which is
the convention the library uses for its own capsules, and the code disagrees with the name
everywhere. A rebuild should name it Y and move on.

## the shape kind

**Contract** — the first cylinder created registers a new shape kind with the dynamics
library, supplying its byte size, its collision-function lookup, its extent routine and no
destructor (a cylinder owns nothing). The identifier lives in one module-level slot.

Creating a cylinder allocates the shape, adds it to a collision space if one is given, and
stores the radius and length. Setting and getting the parameters check the shape's kind first.

**Notes** — the lazy once-only registration and the global identifier are the library's idiom
for user shapes; in a rebuild with open collision dispatch this ceremony disappears.

## extent

**Contract** — the world-axis-aligned extent of an oriented cylinder, exactly, per axis:

```text
FOR EACH world axis a
  half_extent[a] = |axis_row[a]| * length/2          # the axis' contribution
                 + sqrt(other_two_rows[a] squared)   # the disc's contribution
                 * radius
```

**Notes** — the second term is the length of the projection of the cylinder's *disc plane*
onto the world axis, which is exactly the radius times the sine of the angle between the
cylinder axis and that world axis. Writing it as a square root of two squared components
avoids a trigonometric call and is exact; a bounding sphere would be simpler and would cost
the broad phase a large fraction of its pruning.

## cylinder versus box

**Contract** — report whether an oriented cylinder and an oriented box overlap, and if so
produce a separating normal, a penetration depth, a code naming which axis separated them
least, and one to three contact points.

The axis set is the heart of it. Eleven candidate axes are tested, in this order, and any one
of them that separates the pair ends the test immediately with "no overlap":

```text
  code 0      the cylinder's own axis                    # a box vertex on a flat end
  codes 1..3  the box's three face normals               # a cylinder rim on a box face
  code 4      the normal from the cylinder axis to the
              box's nearest vertex                       # a box vertex on the barrel
  codes 5..7  for each box axis: the cross of that axis
              with the rim tangent at the point the
              cylinder's ring faces it                   # a cylinder RING on a box EDGE
  codes 8..10 the cross of the cylinder axis with each
              box axis                                   # barrel against edge
```

For each axis the test is the standard one — project the half-extent of both shapes onto the
axis, compare against the projected centre distance — and the shape-specific part is how the
cylinder's half-extent is computed:

```text
cylinder half-extent along axis a
  = |cos| * length/2 + sin * radius
  where cos = dot(a, cylinder_axis),  sin = sqrt(1 - cos²)
```

The axis with the **smallest** positive penetration wins, and its depth and normal become the
result. For the cross-product axes the depth must be divided by the axis' length before
comparison, because those axes are not normalised; failing to do so systematically prefers
long cross products and picks the wrong normal.

**Invariants** — an axis that separates returns "no overlap" immediately. An axis whose cross
product degenerates to (near) zero length must be skipped, not normalised, because the two
shapes' axes are parallel and that axis carries no information.

**Notes** — codes 5 through 7 are the part a rebuilder will not think of. The rim of a
cylinder meeting the edge of a box is not separated by any of the obvious axes, and getting it
wrong makes a cylinder sink into box corners. Constructing that axis is three steps: find the
direction from the box axis to the cylinder centre, rotate it ninety degrees within the
cylinder's disc plane to get the rim tangent at the facing point, then cross that tangent with
the box axis.

**Contact synthesis** depends on which code won:

- **cross-product axes (8–10)** — an edge-to-edge touch: find the deepest point on the
  cylinder's rim along the normal and the deepest box vertex, then take the closest approach
  of the two lines through them. One contact.
- **code 4** — a single contact at the box vertex already computed while building the axis.
- **code 0** — a single contact at the box's deepest vertex along the normal.
- **codes 1–3** — a flat end of the cylinder against a box face. One contact at the deepest
  rim point, plus *two more* at ±60° around the rim from it, each accepted only if its own
  depth is still positive. This is the manifold that stops a standing cylinder from tipping.

**Invariants** — the three-point manifold is generated only when the cylinder's disc is nearly
parallel to the box face (tested as the sine of the angle being below `1/√2`, i.e. 45°).
Beyond that angle the extra points are behind the surface and would push the cylinder the
wrong way.

**Notes** — ±60° is the choice that matters: three points evenly spaced around a circle is the
minimum that constrains both tipping axes, and placing the first at the deepest point makes
the set symmetric about the steepest direction. Two points would leave a free rocking axis;
four would cost a quarter more solver work for a constraint that is already redundant.

## cylinder versus cylinder

**Contract** — the same technique with a different axis set:

```text
  code 0      the first cylinder's axis
  code 6      the cross of the two axes
  code 3      the normal from cylinder 1's axis to
              cylinder 2's deepest point
  code 4      the mirror of that, from 2 to 1
  code 5      the axis through the intersection region of
              the two rim circles
```

Contact synthesis: the cross-axis case takes the closest approach of the two rims; codes 3, 4
and 5 each take the single point that built them; code 0 and the remaining face cases produce
the same ±60° three-point rim manifold as the box case, generated on whichever cylinder's flat
face is involved.

**Notes** — code 5 is the rim-against-rim case, and it needs a genuinely separate construction:
*circle intersection*. Two rim circles lie in two planes; the planes meet in a line; each
circle's plane-line intersection gives two points; the contact is the midpoint of the closest
such pair. When the circles do not actually reach the line the routine **deliberately takes the
imaginary root's magnitude instead of failing**, with the original noting this is "somewhat
strange" but that some axis must be produced to separate the edges as they approach. That is a
real decision and not a bug: returning "no axis" there lets two rims interpenetrate.

The comment in the original notes the face-to-face case is approximated and would want a
proper manifold. It shows as two cylinders stacked flat on each other wobbling slightly.

## cylinder versus sphere

**Contract** — three axes only: the cylinder's axis, the perpendicular from that axis to the
sphere's centre, and the direction from the sphere's centre to the deepest rim point. The
winner gives the normal; the contact point is the sphere's centre pushed back along the normal
by its radius. Always one contact — a sphere touches at a point by definition.

## cylinder versus plane

**Contract** — no axis search: a plane has one normal. The depth is the cylinder's half-extent
along the plane normal minus the centre's signed distance; negative means no contact. The
first contact is the single deepest point on the cylinder surface. Then, exactly as in the box
case, the manifold is completed by the cylinder's orientation:

```text
IF the cylinder's axis is within 45° of the plane normal    # standing on an end
  add two more rim points, at ±90° in the disc's two principal directions
ELSE                                                        # lying on its side
  add one more point at the OTHER end of the cylinder
```

Each extra point is accepted only if its own depth is positive.

**Notes** — this is the clearest statement in the file of what a manifold is *for*. A cylinder
standing upright needs three points to stop it tipping in two directions; a cylinder lying
down needs two, one per end, to stop it rotating about the contact. Producing the same
manifold for both configurations would be wrong in both.

## cylinder versus ray

**Contract** — the standard ray-versus-finite-cylinder intersection, cased carefully:

```text
FUNCTION cylinder_vs_ray(cylinder, ray) -> optional<contact>
  project the ray start onto the cylinder's axis and its disc plane
  inside := the start is within both the radius and the two caps

  IF the start is inside the INFINITE cylinder but outside the caps
    test only the near cap                       # it can only enter through an end
  ELSE
    solve the quadratic for the infinite barrel
    IF no real root
      IF not inside THEN RETURN none
      test the cap the ray is heading toward     # inside and parallel to the axis
    ELSE
      take the nearer non-negative root; reject if beyond the ray's length
      IF the hit lies between the caps
        normal := the outward radial direction there, flipped if the ray started inside
        RETURN contact at that point, depth = distance along the ray
      otherwise test the cap on that side

  # cap test: intersect the ray with the cap's plane
  reject if parallel, if behind the start, or if beyond the ray's length
  normal := the cap's outward axis direction
  RETURN contact at the plane point
```

**Invariants** — the reported **depth is the distance along the ray**, not a penetration
depth. Every other routine in this file reports penetration. A ray is a probe, not a solid, so
the field carries the hit distance instead; consumers must know which they are holding.

**Notes** — the cap test is performed without re-checking that the hit lies within the cap's
radius. In the path that reaches it that check is implied by the barrel test having failed,
but it is not obvious, and a rebuild that restructures the cases must re-add it explicitly.

The ray case is not in the shape's own dispatch table; it is reached only through the motion
probe ([`../dRayMotions.cpp`](../dRayMotions.cpp.md)), which calls it directly and then
reverses the contact because the argument order is forced.

## the dispatch table

**Contract** — a cylinder answers collisions against boxes, spheres, cylinders and planes.
Every case returns contacts with the cylinder as the **second** shape and the normal negated
from the internal convention, because the internal routines compute the normal as "the
direction to separate the cylinder along" while the library's convention is the opposite.

**Notes** — the negation appears once per case, mechanically, and is exactly the sort of
convention mismatch a rebuild should settle once at the boundary rather than at five call
sites. Triangle meshes are absent from this table on purpose: that pairing is owned by the
mesh collider's own dispatch
([`../tri-colliderknoopc/dTriList.cpp`](../tri-colliderknoopc/dTriList.cpp.md)), which
registers the cylinder case from its side.
