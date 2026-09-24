# src/xrPhysics/tri-colliderknoopc/dcTriangle.h

> One triangle, pre-chewed: its two edges, its plane, and how far the shape being tested is
> from that plane.

**Needs** — [`../../xrCDB/xrCDB.h`](../../xrCDB/xrCDB.h.md)
**Used by** — [`dTriColliderMath.h`](dTriColliderMath.h.md) · [`dcTriListCollider.h`](dcTriListCollider.h.md)
**Tier floor** — T2: a derived-values record.

## Purpose

The mesh collider touches the same triangle several times per step — once for the plane test,
once for the containment test, once per contact — and each touch needs the same handful of
derived quantities. Computing them once into this record and passing it around is the whole
point; it is not a data structure decision but a "compute it once" decision, and it is worth
half the collider's running time.

## State

```text
RECORD PreparedTriangle
  edge_0   : vector        # vertex1 - vertex0
  edge_1   : vector        # vertex2 - vertex1
  normal   : vector        # normalized cross(edge_0, edge_1)
  plane_d  : real          # dot(vertex0, normal) — the plane's offset
  distance : real          # signed distance of the TESTED SHAPE's centre from
                           # that plane; negative means the shape is behind the
                           # triangle's face
  depth    : real          # how deep the shape penetrates along the normal
  source   : reference to the collision database's triangle record
```

**Invariants** — the normal is normalised and the third edge is never stored: it is
`-(edge_0 + edge_1)` and every consumer that needs it recomputes it. Winding order determines
the normal's sign, so the normal points out of the *front* face and `distance < 0` means the
shape is behind the triangle. The whole collider's logic turns on that sign, so the collision
database's triangles must be consistently wound.

The `source` reference points into the level's static triangle array and is borrowed, never
owned. The record is scratch: it lives for one shape's test against one triangle.

**Notes** — the record deliberately does *not* hold the three vertices. They are looked up
from the shared vertex array by the triangle's indices at each use site, because the collider
needs the *indices* as well (to recognise shared edges and vertices between adjacent triangles
— see [`dcTriListCollider.h`](dcTriListCollider.h.md)) and carrying both would be redundant.

`depth` and `distance` are initialised to negative infinity in checked builds only, so that
reading either before it has been set is catchable. In a rebuild they are simply not readable
until set.
