# src/xrPhysics/tri-colliderknoopc/dTriCylinder.h

> The cylinder's record, and its reach along an arbitrary direction — the one number the mesh
> traversal needs from it.

**Needs** — [`../dcylinder/dCylinder.h`](../dcylinder/dCylinder.h.md) · [`dcTriListCollider.h`](dcTriListCollider.h.md) · [`TriPrimitiveCollideClassDef.h`](TriPrimitiveCollideClassDef.h.md) · [`dTriCylinder.cpp`](dTriCylinder.cpp.md)
**Used by** — [`dCylinder.cpp`](../dcylinder/dCylinder.cpp.md) · [`dTriCollideK.h`](dTriCollideK.h.md) · [`dTriCylinder.cpp`](dTriCylinder.cpp.md)
**Tier floor** — T1: it declares the byte layout the dynamics library stores inside a
user-shape's payload.

## Purpose

Two things: the cylinder's stored fields, and its projection.

## State

```text
RECORD Cylinder
  radius : real
  length : real      # full length along the shape's LOCAL Y axis
```

**Invariants** — this record is the payload the dynamics library allocates inside the shape
when the cylinder kind is registered, so its size is declared to the library at registration
time ([`../dcylinder/dCylinder.cpp`](../dcylinder/dCylinder.cpp.md)) and the two must agree.
It is repeated here rather than shared because the mesh collider is compiled separately from
the cylinder's own collision code — a duplication a rebuild should not reproduce.

## cylinder projection

**Contract** — how far a cylinder reaches from its centre along a given unit direction.

```text
FUNCTION cylinder_reach(cylinder, direction) -> real
  cos := |dot(direction, cylinder.axis)|
  cos := min(cos, 1)                       # guard: it may exceed 1 by rounding
  sin := sqrt(1 - cos²)
  RETURN cos * (length / 2) + sin * radius
```

**Invariants** — the clamp before the square root is not decoration. `cos` is a dot product of
two vectors each of which is only approximately unit length, so it can exceed one by a few
units in the last place; without the clamp the square root takes a negative argument and the
reach becomes not-a-number, which propagates into a contact depth and from there into the
solver, where it destroys the whole island. This is a real failure mode and the guard is
repeated at every site in the chapter that takes this particular square root.

**Notes** — this is the sole thing the mesh traversal asks of the cylinder, and its exactness
matters: the traversal compares reaches from different triangles to decide which one pushes
the shape out, so an over-estimate changes the chosen normal rather than merely being
conservative. The same formula appears in the cylinder's own collision routines and in its
extent computation, which is the honest sign that it should be one named function.
