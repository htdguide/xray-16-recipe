# src/xrPhysics/tri-colliderknoopc/dTriSphere.h

> Declares the sphere's adaptor into the mesh traversal.

**Needs** — [`TriPrimitiveCollideClassDef.h`](TriPrimitiveCollideClassDef.h.md) · [`../MathUtils.h`](../MathUtils.h.md) · [`dTriSphere.cpp`](dTriSphere.cpp.md)
**Used by** — [`dTriCollideK.h`](dTriCollideK.h.md) · [`dTriSphere.cpp`](dTriSphere.cpp.md)
**Tier floor** — T4: two includes.

## Purpose

The sphere's half of the primitive interface needs no declarations of its own: its reach along
any direction is its radius, which is short enough to live inline in
[`dcTriListCollider.h`](dcTriListCollider.h.md), and everything else is in
[`dTriSphere.cpp`](dTriSphere.cpp.md). The file exists so the three primitives have
symmetrical headers.

## Stateless.
