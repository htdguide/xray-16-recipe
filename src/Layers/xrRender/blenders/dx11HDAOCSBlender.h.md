# src/Layers/xrRender/blenders/dx11HDAOCSBlender.h

> Declares the compute-shader ambient-occlusion templates.

**Needs** — [`Blender.h`](../Blender.h.md) · [`dx11HDAOCSBlender.cpp`](dx11HDAOCSBlender.cpp.md)
**Used by** — [`dx11HDAOCSBlender.cpp`](dx11HDAOCSBlender.cpp.md)
**Tier floor** — T2: a declaration only.

## Purpose

Declares the surface implemented in [`dx11HDAOCSBlender.cpp`](dx11HDAOCSBlender.cpp.md).

Exported units:

- **`CBlender_CS_HDAO`** — internal template, no class identifier; one compute pass.
- **`CBlender_CS_HDAO_MSAA`** — the same for a multisampled g-buffer; the program iterates samples itself rather than being compiled per sample.

Neither is detailable or lightmappable, and neither carries a constructor: they have no parameters and no identity, so the base class's zero-initialized description stands.
