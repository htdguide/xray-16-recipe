# src/Layers/xrRenderDX11/3DFluid/dx113DFluidVolume.h

> Declares the fluid volume's scene-object face.

**Needs** — [`dx113DFluidVolume.cpp`](dx113DFluidVolume.cpp.md) · [`dx113DFluidData.h`](dx113DFluidData.h.md) · [`xrRender/FBasicVisual.h`](../../xrRender/FBasicVisual.h.md)
**Used by** — [`dx113DFluidVolume.cpp`](dx113DFluidVolume.cpp.md)
**Tier floor** — T2: a scene-graph node.

## Purpose

Declares the surface implemented in [`dx113DFluidVolume.cpp`](dx113DFluidVolume.cpp.md): a visual that owns one fluid volume's data and whose draw method simulates and renders it.

## Exported units

- **the fluid visual** — `Load` from the level's record, `Render`, `Copy`, `Release`, all overriding the base visual's contract.
- it holds the volume's data record and a debug geometry handle.
