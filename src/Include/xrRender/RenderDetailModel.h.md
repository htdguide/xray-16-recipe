# src/Include/xrRender/RenderDetailModel.h

> An opaque handle to one detail-object model — a tuft of grass or a piece of debris — with no surface at all.

**Needs** — [`RenderVisual.h`](RenderVisual.h.md)
**Used by** — [`IRenderDetailModel.h`](../../Layers/xrRender/IRenderDetailModel.h.md) · [`dxRainRender.cpp`](../../Layers/xrRender/dxRainRender.cpp.md) · [`dxThunderboltDescRender.cpp`](../../Layers/xrRender/dxThunderboltDescRender.cpp.md) · [`dxThunderboltDescRender.h`](../../Layers/xrRender/dxThunderboltDescRender.h.md) · [`dxThunderboltRender.cpp`](../../Layers/xrRender/dxThunderboltRender.cpp.md)
**Tier floor** — T3: a name for a type; nothing here constrains anything.

## Purpose

The *detail objects* layer is the grass and small debris scattered across a level's ground. It is not made of ordinary models: a level's detail data is a fixed palette of a few dozen small meshes plus a per-square-metre density and slot map, and the renderer expands it into geometry per frame around the camera, with its own culling, fading and wind animation. All of that is the renderer's, none of it is the engine's.

This file exists so the engine can *name* one of those palette entries — in a function signature, in a field — without knowing anything about it.

## State

`Stateless.`

## `IRenderDetailModel`

**Contract** — an empty interface. It declares nothing, demands nothing of an implementor, and has no methods. An implementor is whatever the renderer's detail palette holds.

**Notes** — This is a placeholder that outlived its purpose. The engine once created detail models from a stream through the renderer's model factory and held handles to them; that call and every caller are gone, commented out on the renderer interface. What remains is the type name with no users of substance.

A rebuild deletes the file. It is recorded here because the mirror must be complete, and because its emptiness is itself the finding: **the detail-object layer has no engine-side surface at all**. The engine loads the level, and the renderer reads the detail data out of the same level directory on its own. If a rebuild wants the engine to have any say over grass — density, radius, whether it exists — that seam has to be invented, because it does not exist today.
