# src/xrPhysics/tri-colliderknoopc/TriPrimitiveCollideClassDef.h

> Binds one primitive's three operations into a value the shared mesh traversal can be
> instantiated over.

**Needs** — [`dcTriListCollider.h`](dcTriListCollider.h.md)
**Used by** — [`dTriBox.h`](dTriBox.h.md) · [`dTriCylinder.h`](dTriCylinder.h.md) · [`dTriSphere.h`](dTriSphere.h.md) · [`dcTriListCollider.h`](dcTriListCollider.h.md)
**Tier floor** — T2: a dispatch adaptor.

## Purpose

The mesh traversal in [`dSortTriPrimitive.h`](dSortTriPrimitive.h.md) is written once and used
by three primitives. It needs, from each, three things: the primitive's reach along a
direction, its ordinary collision against a triangle, and its recovery collision against a
triangle's plane. This file declares and defines the three-method adaptor, once, for whichever
primitive name it is given.

That is all it is. In the original it is a pair of macros generating a class and its method
bodies, because the three operations live as differently named methods on one large collider
object and there is no other way to make them interchangeable without a virtual call the
traversal cannot afford.

## Stateless.

## the primitive adaptor

**Contract** — for each of box, sphere and cylinder there is a small value holding a reference
to the collider, exposing:

```text
  Proj(shape, normal) -> real
  Collide(v0, v1, v2, prepared_triangle, shape, mesh, flags, contacts, stride) -> int
  CollidePlain(edge_0, edge_1, normal, triangle, distance,
               shape, mesh, flags, contacts, stride) -> int
```

Each forwards to the correspondingly named method on the collider. The adaptor is not
copyable — it holds a reference — and is constructed fresh at each entry point.

**Notes** — the whole file is incidental. The decision it encodes — *the traversal is
parametrised over the primitive, and the primitive contributes exactly three operations* — is
load-bearing and is stated in [`dcTriListCollider.h`](dcTriListCollider.h.md); the means by
which it is encoded is not. A rebuild writes three small types implementing one interface, or
three closures, or three cases of a sum type, and this file has no successor.

What must survive is that the dispatch is resolved at compile time and not through an indirect
call. The traversal calls `Proj` several times per candidate triangle across a few hundred
candidates per shape per step, and an indirect call there is measurable.
