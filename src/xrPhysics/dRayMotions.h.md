# src/xrPhysics/dRayMotions.h

> Declares the swept-motion probe shape: a ray that reports hits as if it were the body it
> belongs to.

**Needs** — [`dRayMotions.cpp`](dRayMotions.cpp.md) · [Seam: Rigid-body dynamics](../../SYSTEM-REQUIREMENTS.md#seam-rigid-body-dynamics)
**Used by** — [`PHObject.cpp`](PHObject.cpp.md) · [`PHWorld.cpp`](PHWorld.cpp.md) · [`dRayMotions.cpp`](dRayMotions.cpp.md)
**Tier floor** — T1: it registers a new shape kind with the dynamics library, which is a
layout-and-vtable contract with that library.

## Purpose

Declares the surface implemented in [`dRayMotions.cpp`](dRayMotions.cpp.md).

Exported units:

- **the shape-kind identifier**, assigned once when the first probe is created;
- **create a probe** in a collision space;
- **aim a probe** — origin, direction and length;
- **name the probe's owner** — the shape whose identity every hit will be reported under.
