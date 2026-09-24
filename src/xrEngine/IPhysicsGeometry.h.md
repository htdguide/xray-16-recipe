# src/xrEngine/IPhysicsGeometry.h

> One collision shape, asked only for its oriented bounding box and whether water touches it.

**Needs** — _none_
**Used by** — [`dx113DFluidObstacles.cpp`](../Layers/xrRenderDX11/3DFluid/dx113DFluidObstacles.cpp.md) · [`IPhysicsShell.h`](IPhysicsShell.h.md) · [`Geometry.h`](../xrPhysics/Geometry.h.md)
**Tier floor** — T3: two accessors

## Purpose

An object's physical shell is a tree of elements, each carrying several primitive shapes.
Outside the physics module almost nothing wants the shape itself — it wants a box to draw,
to cull against, or to test for immersion. This interface is that reduced view, and it is
deliberately the smallest one that still answers the two questions non-physics code asks.

## State

`Stateless.`

## `IPhysicsGeometry`

**Contract** — `get_box` fills a transform and a half-extent triple describing the shape's
oriented bounding box in world space at the current simulation pose; a sphere reports an
axis-aligned box of equal extents. `collide_fluids` says whether the shape participates in
the water/immersion test — most do, some (sensor volumes, non-colliding proxies) do not.
Neither call allocates or blocks; both are safe to call between physics steps only.

```text
INTERFACE PhysicsGeometry
  get_box() -> (form : matrix4, half_extents : vector3)   # oriented, world space
  collide_fluids() -> bool
```
