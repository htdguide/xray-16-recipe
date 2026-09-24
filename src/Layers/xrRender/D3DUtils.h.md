# src/Layers/xrRender/D3DUtils.h

> Declares the shape-drawing toolbox: the ten preloaded primitive meshes, the three dynamic vertex layouts, and the several dozen debug shapes built on them.

**Needs** — [`Include/xrRender/DrawUtils.h`](../../Include/xrRender/DrawUtils.h.md) · [`BufferUtils.h`](BufferUtils.h.md) · [`FVF.h`](FVF.h.md) · [`R_DStreams.h`](R_DStreams.h.md)
**Used by** — [`D3DUtils.cpp`](D3DUtils.cpp.md) · [`D3DXRenderBase.cpp`](D3DXRenderBase.cpp.md) · [`xrRender_R4.cpp`](../xrRenderPC_R4/xrRender_R4.cpp.md)
**Tier floor** — T1: it declares device buffers and a raw pointer walking a mapped vertex range.

## Purpose

Declares the surface implemented in [`D3DUtils.cpp`](D3DUtils.cpp.md): the renderer's filling of the shape-drawing interface the engine, the physics debug layer and the editors all reach through.

## Exported units

- **`PrimitiveBuffer`** — one preloaded shape: its own vertex and index buffers, its primitive kind and count, and a bound draw operation chosen at build time depending on whether it is indexed.
- **`DrawUtilities`** — the interface's implementation. Holds ten primitive buffers (solid and wireframe variants of box, sphere, sphere cap, cone and cylinder), three dynamic-stream geometry layouts, a debug font, and the accumulating triangle batch.
- **`DUImpl`** — the single global instance. The interface has exactly one implementation and one instance, reached through a global; see [Seam notes in the chapter README](README.md).

**Notes** — The class inherits both the shape interface and the engine's per-frame render callback, because it must flush its font at a fixed point in the frame. That double inheritance is incidental; what survives is "this object must be given a per-frame tick".
