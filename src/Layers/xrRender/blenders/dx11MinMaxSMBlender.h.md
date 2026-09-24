# src/Layers/xrRender/blenders/dx11MinMaxSMBlender.h

> Declares the shadow-map min/max reduction template.

**Needs** — [`Blender.h`](../Blender.h.md) · [`dx11MinMaxSMBlender.cpp`](dx11MinMaxSMBlender.cpp.md)
**Used by** — [`dx11MinMaxSMBlender.cpp`](dx11MinMaxSMBlender.cpp.md) · [`glMinMaxSMBlender.cpp`](glMinMaxSMBlender.cpp.md) · [`r3_rendertarget_create_minmaxSM.cpp`](../../xrRender_R2/r3_rendertarget_create_minmaxSM.cpp.md)
**Tier floor** — T2: a declaration only.

## Purpose

Declares the surface implemented in [`dx11MinMaxSMBlender.cpp`](dx11MinMaxSMBlender.cpp.md), and — because the class is shared — in [`glMinMaxSMBlender.cpp`](glMinMaxSMBlender.cpp.md). Exactly one of those two files is built.

Exported units:

- **`CBlender_createminmax`** — internal template, no class identifier, neither detailable nor lightmappable; emits the min/max reduction pass.
