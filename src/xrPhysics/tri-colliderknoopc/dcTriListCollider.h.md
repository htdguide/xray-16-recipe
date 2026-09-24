# src/xrPhysics/tri-colliderknoopc/dcTriListCollider.h

> The mesh collider's shape: what it caches between steps, and the per-triangle flags that stop
> a shape being pushed twice by one shared edge.

**Needs** — [`dcTriangle.h`](dcTriangle.h.md) · [`TriPrimitiveCollideClassDef.h`](TriPrimitiveCollideClassDef.h.md) · [`dcTriListCollider.cpp`](dcTriListCollider.cpp.md) · [`dSortTriPrimitive.h`](dSortTriPrimitive.h.md) · [`../../xrCDB/xrCDB.h`](../../xrCDB/xrCDB.h.md) · [Seam: Static collision database](../../../SYSTEM-REQUIREMENTS.md#seam-static-collision-database)
**Used by** — [`TriPrimitiveCollideClassDef.h`](TriPrimitiveCollideClassDef.h.md) · [`dSortTriPrimitive.h`](dSortTriPrimitive.h.md) · [`dTriBox.cpp`](dTriBox.cpp.md) · [`dTriBox.h`](dTriBox.h.md) · [`dTriCylinder.cpp`](dTriCylinder.cpp.md) · [`dTriCylinder.h`](dTriCylinder.h.md) · [`dTriList.cpp`](dTriList.cpp.md) · [`dTriSphere.cpp`](dTriSphere.cpp.md) · [`dcTriListCollider.cpp`](dcTriListCollider.cpp.md) · [`dxTriList.h`](dxTriList.h.md)
**Tier floor** — T1: it holds iterators into a shared triangle-index list and writes contacts
into the solver's array.

## Purpose

This is the declaration of the object that answers "this shape is here; what of the level does
it touch?". The three public entry points — box, sphere, cylinder — live in
[`dcTriListCollider.cpp`](dcTriListCollider.cpp.md); the traversal they share lives in
[`dSortTriPrimitive.h`](dSortTriPrimitive.h.md); the per-primitive contact generation lives in
[`dTriBox.cpp`](dTriBox.cpp.md), [`dTriSphere.cpp`](dTriSphere.cpp.md) and
[`dTriCylinder.cpp`](dTriCylinder.cpp.md).

What is decided *here* is the **edge-and-vertex engagement bookkeeping**, which is the single
idea that makes a triangle-soup collider usable at all, and which nothing else in the recipe
explains.

## State

```text
RECORD MeshCollider
  shape          : Shape                 # the mesh shape this collider serves
  positive_tries : list<PreparedTriangle>  # triangles the shape is in FRONT of,
                                           # kept for one traversal only
  negative_tries : list<PreparedTriangle>  # (declared; the traversal uses locals)
  engagement     : list<bit set>         # one entry per CANDIDATE TRIANGLE this step
  begin, current, end : iterators into the candidate triangle index list
```

```text
ENUM engagement_bit
  vertex_0, vertex_1, vertex_2     # this triangle's vertex N has already produced
                                   # a contact, via some other triangle
  side_0, side_1, side_2           # this triangle's edge N likewise
```

**Invariants** — `engagement` is indexed by position in the candidate list, not by triangle
id, and is resized and cleared at the start of each traversal. The iterators are members
rather than parameters because the engagement marking routines need to walk *forward from the
current triangle* to the end, marking the triangles that have not been visited yet.

**Notes** — the iterators-as-members are the file's ugliest decision and its most revealing.
They exist because marking engagement is a side effect that reaches forward into the rest of
the traversal, and the alternative — threading the position through six call levels — was
worse. In a rebuild, engagement marking is a method on the traversal state and this disappears.

## the shared-feature problem, and what engagement solves

**Contract** — a triangle soup has no topology. Two triangles that share an edge are two
independent triangles, and a sphere resting on that edge is found to be touching *both*. Each
produces a contact, with different normals, and the solver pushes the sphere twice — so it
jumps, or skids along a flat floor that happens to be tessellated.

The fix is: **the first triangle to produce a contact from a shared feature claims it, and
marks every later candidate that shares it.**

```text
FUNCTION claim_vertex(v, triangle_array)
  FOR EACH later candidate L IN (current + 1 .. end)
    FOR EACH i IN 0..2
      IF triangle_array[L].vertex[i] == v
        engagement[L].set(vertex_i)

FUNCTION claim_edge(v0, v1, triangle_array)
  FOR EACH later candidate L IN (current + 1 .. end)
    IF L has an edge running from v1 to v0        # note: REVERSED
      engagement[L].set(the corresponding side bit)
```

A primitive's contact generator consults the current triangle's engagement bits before
producing a vertex or edge contact, and declines if the feature is already claimed
(see [`dTriSphere.cpp`](dTriSphere.cpp.md) and [`dTriCylinder.cpp`](dTriCylinder.cpp.md)).

**Invariants** — the edge match is on the **reversed** winding. Two consistently wound
triangles sharing an edge traverse it in opposite directions, so `(a→b)` on one is `(b→a)` on
the other. Matching the same direction finds nothing, and matching either direction would
wrongly join two triangles that face away from each other across a fold.

Claiming only looks *forward*. Triangles already visited are not marked, which is correct
because they have already had their chance and either produced a contact or did not.

**Notes** — the engagement set is per traversal and per shape, so two shapes resting on the
same edge each get one contact, which is right. It is also why the engagement list is a member
of the collider rather than of the shape: one collider serves one mesh, and traversals for
different shapes are serialised.

The face case needs no engagement bit: a point is over at most one triangle's face, so face
contacts are naturally unique. Only edges and vertices are shared, which is exactly why there
are six bits and not nine.

## the per-primitive interface

**Contract** — the traversal is written once and instantiated for three primitives. Each
primitive supplies exactly three operations:

```text
  Proj(shape, normal) -> real
      # the shape's half-extent along that direction: how far it reaches. This is
      # what turns a plane distance into a penetration depth.

  Collide(v0, v1, v2, prepared_triangle, ...) -> contact count
      # the shape against a triangle it is in FRONT of: full face/edge/vertex
      # treatment, consulting and claiming engagement

  CollidePlain(edge_0, edge_1, normal, triangle, distance, ...) -> contact count
      # the shape against the PLANE of a triangle it is BEHIND: no containment
      # test, no engagement — just push it out along the normal
```

**Invariants** — `Proj` must be exact, not a bound. The traversal compares depths from
different triangles to choose which one pushes the shape out, so a conservative over-estimate
on one primitive silently changes which triangle wins.

**Notes** — the two collision forms are the recovery/normal split. `Collide` is the ordinary
case; `CollidePlain` is what runs when the shape has already ended up *inside* the level and
must be extracted. The second cannot use containment or engagement, because a shape inside the
geometry is behind many triangles at once and the point is to pick one and push hard.

## `CollideBox`, `CollideSphere`, `CollideCylinder`

**Contract** — implemented in [`dcTriListCollider.cpp`](dcTriListCollider.cpp.md).
