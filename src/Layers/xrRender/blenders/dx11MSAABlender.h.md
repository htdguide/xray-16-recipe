# src/Layers/xrRender/blenders/dx11MSAABlender.h

> Declares the multisample edge-marking template.

**Needs** — [`Blender.h`](../Blender.h.md) · [`dx11MSAABlender.cpp`](dx11MSAABlender.cpp.md)
**Used by** — [`dx11MSAABlender.cpp`](dx11MSAABlender.cpp.md) · [`glMSAABlender.cpp`](glMSAABlender.cpp.md) · [`r3_rendertarget_mark_msaa_edges.cpp`](../../xrRender_R2/r3_rendertarget_mark_msaa_edges.cpp.md)
**Tier floor** — T2: a declaration only.

## Purpose

Declares the surface implemented in [`dx11MSAABlender.cpp`](dx11MSAABlender.cpp.md), and — because the class is shared — in [`glMSAABlender.cpp`](glMSAABlender.cpp.md). Exactly one of those two files is built.

Exported units:

- **`CBlender_msaa`** — internal template, no class identifier, neither detailable nor lightmappable; emits the edge-marking pass.
