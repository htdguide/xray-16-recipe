# src/xrPhysics/tri-colliderknoopc/dcTriListCollider.cpp

> The three doors into the mesh collider, and the one thing they each decide: how big a region
> of the level to ask about.

**Needs** — [`dcTriListCollider.h`](dcTriListCollider.h.md) · [`dSortTriPrimitive.h`](dSortTriPrimitive.h.md) · [`dTriCollideK.h`](dTriCollideK.h.md) · [`dxTriList.h`](dxTriList.h.md) · [`../dcylinder/dCylinder.h`](../dcylinder/dCylinder.h.md) · [`../MathUtils.h`](../MathUtils.h.md) · [`../../xrCDB/Intersect.hpp`](../../xrCDB/Intersect.hpp.md) · [Seam: Static collision database](../../../SYSTEM-REQUIREMENTS.md#seam-static-collision-database)
**Used by** — [`dTriList.cpp`](dTriList.cpp.md) · [`dcTriListCollider.h`](dcTriListCollider.h.md)
**Tier floor** — T1: it reads shape parameters and body velocities out of the dynamics
library's records.

## Purpose

Three entry points, one per primitive, each doing the same two things: compute the shape's
axis-aligned half-extent, inflate it by how far the shape will travel this step, and hand both
to the shared traversal ([`dSortTriPrimitive.h`](dSortTriPrimitive.h.md)).

That inflation is the file's content. Everything else is per-shape extent arithmetic.

Not a translation unit of its own: it is included into
[`dTriList.cpp`](dTriList.cpp.md) so these three can be inlined into the dispatch.

## Stateless.

## the velocity margin

**Contract** — every entry point adds, to each component of the half-extent, that component of
the body's linear velocity times a fixed 0.04 seconds.

**Invariants** — the margin is added **per component and unsigned**, so the box grows in both
directions along every axis rather than extending only forward. That is deliberate: the box is
used both to query the collision database and as the region within which the swept test
operates, and the swept test needs the shape's *previous* position inside it as well as its
next.

**Notes** — 0.04 seconds is not a tuning knob; it is two fixed steps at the engine's
simulation rate. One step would be the minimum that sees where the shape is going; two gives a
step of slack so the cached candidate set survives a step without being re-queried. That
caching is the point — the traversal keeps the previous query's result and reuses it as long
as the new box still fits inside the old one — and the margin is what makes the reuse rate
high. Shrink it and query count rises; grow it and every query returns more triangles than it
needs.

The constant is written literally at each of the three sites rather than named once, which is
the kind of thing a rebuild should fix by naming it after what it is: *two steps of lookahead*.

## `CollideBox`

**Contract** — the box's world-axis-aligned half-extent is the sum over its three local axes
of that axis' contribution, plus a small fixed epsilon, plus the velocity margin. The velocity
is read only if the box has a body at all.

```text
half_extent[a] = ( Σ over local axes i of |side[i] * rotation[a][i]| ) / 2
               + 10 * small_epsilon
               + |velocity[a]| * 0.04
```

**Notes** — the box path is the only one that checks for a body before reading its velocity,
because a box may be attached to static geometry (a door frame, a placed prop) while a sphere
or cylinder in this engine always belongs to something that moves. The sphere and cylinder
paths read the velocity unconditionally, which is a latent trap for a rebuild that gives those
shapes to a static object.

The extra epsilon on the box and not on the others is undocumented and looks like the residue
of a specific bug — most likely a box resting exactly on a surface alternating between
touching and not.

## `CollideSphere`

**Contract** — the half-extent is the radius on all three axes, plus the velocity margin. No
orientation term: a sphere has none.

## `CollideCylinder`

**Contract** — the half-extent is the exact oriented-cylinder extent — the axis' contribution
plus the disc's — plus the velocity margin.

```text
half_extent[a] = |rotation[a][axis]| * length/2
               + sqrt(the other two components of row a, squared) * radius
               + |velocity[a]| * 0.04
```

**Notes** — the same formula as the cylinder's own extent routine in
[`../dcylinder/dCylinder.cpp`](../dcylinder/dCylinder.cpp.md), written out a second time. It
is exact rather than a bounding sphere, and it should be: the character controller is a
cylinder, it is the shape that collides with the level most often, and a loose box around it
means querying more of the level every step for every creature in the world.
