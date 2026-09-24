# src/Layers/xrRender/blenders/blender_deffer_model.h

> Declares the model material template and its parameter block.

**Needs** — [`Blender.h`](../Blender.h.md) · [`blender_deffer_model.cpp`](blender_deffer_model.cpp.md)
**Used by** — [`blender_deffer_model.cpp`](blender_deffer_model.cpp.md)
**Tier floor** — T2: a declaration only.

## Purpose

Declares the surface implemented in [`blender_deffer_model.cpp`](blender_deffer_model.cpp.md).

Exported units:

- **`CBlender_deffer_model`** — the template registered under the class identifier `"MODEL   "`. Carries three authored parameters (the alpha-channel flag, the alpha reference, the tessellation selection), answers *yes* to "can be detailed" and *no* to "can be lightmapped" — a dynamic model receives its ambient from the hemisphere term, never from a baked lightmap — and implements the save/load pair and `Compile`.
