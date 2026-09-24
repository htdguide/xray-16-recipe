# src/Layers/xrRenderDX11/3DFluid/dx113DFluidRenderer.h

> Declares the volume renderer: its four intermediate targets, its eight passes, and the single entry point that draws one fluid volume.

**Needs** — [`dx113DFluidRenderer.cpp`](dx113DFluidRenderer.cpp.md)
**Used by** — [`dx113DFluidBlenders.cpp`](dx113DFluidBlenders.cpp.md) · [`dx113DFluidManager.cpp`](dx113DFluidManager.cpp.md) · [`dx113DFluidManager.h`](dx113DFluidManager.h.md) · [`dx113DFluidRenderer.cpp`](dx113DFluidRenderer.cpp.md)
**Tier floor** — T1: it owns render targets with explicit formats and geometry buffers.

## Purpose

Declares the surface implemented in [`dx113DFluidRenderer.cpp`](dx113DFluidRenderer.cpp.md).

## Exported units

- **the intermediate-target enumeration** — ray data at screen resolution, ray data reduced, the marched result, and the edge mask. Four targets is the minimum the reduce-and-refine scheme needs; naming them in an enumeration is what lets the two name tables below stay aligned.
- **`Initialize(width, height, depth)` / `Destroy`** — build and tear down the passes, geometry and generated textures for a grid of this size.
- **`SetScreenSize(width, height)`** — recreate the intermediate targets. Called on window resize; the volume fields are unaffected.
- **`Draw(volume)`** — render one fluid volume into the current frame.
- **the two name tables** — what the engine calls each intermediate target, and what the shader sources call it. Exposed as static queries because [`dx113DFluidBlenders.cpp`](dx113DFluidBlenders.cpp.md) binds them by name at pass-compile time, before any renderer instance exists.

**Notes** — The pass enumeration lists eight entries in two contiguous runs — three ray-data passes, then five raycast passes — and the implementation fills each run from one compiled pass family by index arithmetic. The contiguity is therefore load-bearing: reordering the enumeration silently mis-assigns passes. A rebuild should name each pass rather than index into a family.

The renderer holds a scratch list for its light query as a member, purely to avoid reallocating it per frame.
