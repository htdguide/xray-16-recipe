# src/Layers/xrRender/blenders/blender_light.h

> Declares the forward renderer's additive light template.

**Needs** — [`Blender.h`](../Blender.h.md) · [`blender_light.cpp`](blender_light.cpp.md)
**Used by** — [`blender_light.cpp`](blender_light.cpp.md)
**Tier floor** — T2: a declaration only.

## Purpose

Declares the surface implemented in [`blender_light.cpp`](blender_light.cpp.md).

Exported units:

- **`CBlender_LIGHT`** — the template registered under the class identifier `"LIGHT   "`. No parameters, not lightmappable; implements `Compile` and the human description only.
