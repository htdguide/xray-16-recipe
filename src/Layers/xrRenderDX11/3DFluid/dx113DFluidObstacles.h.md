# src/Layers/xrRenderDX11/3DFluid/dx113DFluidObstacles.h

> Declares the obstacle rasterizer: one entry point that refreshes the occupancy and boundary-velocity fields for one volume.

**Needs** — [`dx113DFluidObstacles.cpp`](dx113DFluidObstacles.cpp.md) · [`dx113DFluidGrid.h`](dx113DFluidGrid.h.md) · [`xrEngine/IPhysicsShell.h`](../../../xrEngine/IPhysicsShell.h.md)
**Used by** — [`dx113DFluidManager.cpp`](dx113DFluidManager.cpp.md) · [`dx113DFluidObstacles.cpp`](dx113DFluidObstacles.cpp.md)
**Tier floor** — T2: the declaration is plain; the device work is in the implementation.

## Purpose

Declares the surface implemented in [`dx113DFluidObstacles.cpp`](dx113DFluidObstacles.cpp.md).

## Exported units

- **`ProcessObstacles(volume, timestep)`** — the only public entry point: refresh both obstacle fields for this volume. The solver calls it once per step, before any step reads them.

Everything else is private: the two pass variants (static box, moving box), the world-to-fluid transform, the spatial query, and the per-shape rasterization.

**Notes** — Three scratch lists are held as members purely to avoid reallocating them every frame — the query result, the multi-part shells and the standalone bodies. The source marks them as wanting a reserved capacity at construction and never gained one. In a rebuild this is a per-frame scratch arena, not object state.
