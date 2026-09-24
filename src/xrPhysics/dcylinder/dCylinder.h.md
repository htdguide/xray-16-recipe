# src/xrPhysics/dcylinder/dCylinder.h

> Declares the cylinder shape the dynamics library does not have.

**Needs** — [`dCylinder.cpp`](dCylinder.cpp.md) · [Seam: Rigid-body dynamics](../../../SYSTEM-REQUIREMENTS.md#seam-rigid-body-dynamics)
**Used by** — [`ExtendedGeom.cpp`](../ExtendedGeom.cpp.md) · [`Geometry.cpp`](../Geometry.cpp.md) · [`Physics.h`](../Physics.h.md) · [`dRayMotions.cpp`](../dRayMotions.cpp.md) · [`dCylinder.cpp`](dCylinder.cpp.md) · [`dTriCylinder.cpp`](../tri-colliderknoopc/dTriCylinder.cpp.md) · [`dTriCylinder.h`](../tri-colliderknoopc/dTriCylinder.h.md) · [`dcTriListCollider.cpp`](../tri-colliderknoopc/dcTriListCollider.cpp.md)
**Tier floor** — T1: registering a shape kind is a layout-and-function-table contract with the
dynamics library.

## Purpose

Declares the surface implemented in [`dCylinder.cpp`](dCylinder.cpp.md).

Exported units:

- **the shape-kind identifier**, assigned once when the first cylinder is created;
- **create a cylinder** in a collision space, from a radius and a length;
- **set** a cylinder's radius and length;
- **get** them back.

**Notes** — the cylinder's axis is its **local Y**, not the Z the name of the length parameter
suggests. That single fact is assumed by every routine in the implementation and by the
triangle-mesh collider's cylinder case
([`../tri-colliderknoopc/dTriCylinder.h`](../tri-colliderknoopc/dTriCylinder.h.md)), and it is
the first thing a rebuilder will get wrong.
