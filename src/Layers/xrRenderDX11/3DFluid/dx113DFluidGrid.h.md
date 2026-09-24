# src/Layers/xrRenderDX11/3DFluid/dx113DFluidGrid.h

> Declares the grid geometry: the four draw calls that cover a three-dimensional field's interior, its boundary, and a flat diagnostic view of it.

**Needs** — [`dx113DFluidGrid.cpp`](dx113DFluidGrid.cpp.md)
**Used by** — [`dx113DFluidEmitters.cpp`](dx113DFluidEmitters.cpp.md) · [`dx113DFluidEmitters.h`](dx113DFluidEmitters.h.md) · [`dx113DFluidGrid.cpp`](dx113DFluidGrid.cpp.md) · [`dx113DFluidManager.cpp`](dx113DFluidManager.cpp.md) · [`dx113DFluidObstacles.cpp`](dx113DFluidObstacles.cpp.md) · [`dx113DFluidObstacles.h`](dx113DFluidObstacles.h.md)
**Tier floor** — T1: it owns device vertex buffers.

## Purpose

Declares the surface implemented in [`dx113DFluidGrid.cpp`](dx113DFluidGrid.cpp.md).

## Exported units

- **`Initialize(width, height, depth)`** — build all four geometries for a grid of this size. Called once; the size never changes.
- **`DrawSlices`** — cover every interior voxel exactly once. The call every simulation step ends with.
- **`DrawBoundaryQuads`** — cover the two end slices.
- **`DrawBoundaryLines`** — cover the four edges of every slice.
- **`DrawSlicesToScreen`** — the diagnostic: every slice laid side by side in one image.
