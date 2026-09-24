# src/xrPhysics/tri-colliderknoopc/dTriList.h

> Declares the mesh shape: how to create one, and the two callbacks it can carry.

**Needs** — [`dTriList.cpp`](dTriList.cpp.md) · [`dxTriList.h`](dxTriList.h.md) · [Seam: Rigid-body dynamics](../../../SYSTEM-REQUIREMENTS.md#seam-rigid-body-dynamics)
**Used by** — [`PHContactBodyEffector.cpp`](../PHContactBodyEffector.cpp.md) · [`Physics.cpp`](../Physics.cpp.md) · [`dTriList.cpp`](dTriList.cpp.md)
**Tier floor** — T1: registering a shape kind with the dynamics library.

## Purpose

Declares the surface implemented in [`dTriList.cpp`](dTriList.cpp.md).

Exported units:

- **the shape-kind identifier**, assigned once when the first mesh shape is created;
- **create a mesh shape** in a collision space, with an optional per-triangle and an optional
  per-batch callback;
- **set and get** each of those two callbacks.

**Notes** — this is a narrowed copy of the declarations in [`dxTriList.h`](dxTriList.h.md):
the same names, minus the payload record and the vector types, for callers that only want to
create a mesh and not to see inside one. The duplication is an artefact; a rebuild has one
declaration.

Note what the creation call does *not* take: any geometry. The mesh shape has no triangles of
its own — it answers every query by asking the level's static collision database
([Seam: Static collision database](../../../SYSTEM-REQUIREMENTS.md#seam-static-collision-database)),
which the rest of the engine already owns. That is the chapter's central arrangement and the
reason this shape exists at all.
