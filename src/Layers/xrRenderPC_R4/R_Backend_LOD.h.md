# src/Layers/xrRenderPC_R4/R_Backend_LOD.h

> Declares the backend's level-of-detail constant channel.

**Needs** — [`R_Backend_LOD.cpp`](R_Backend_LOD.cpp.md) · [`../xrRender/R_Backend.h`](../xrRender/R_Backend.h.md)
**Used by** — [`R_Backend_LOD.cpp`](R_Backend_LOD.cpp.md)
**Tier floor** — T2: a handle and a scalar.

## Purpose

Declares the surface implemented in [`R_Backend_LOD.cpp`](R_Backend_LOD.cpp.md). The submission context embeds one of these, which is why the declaration is separate from the implementation at all.

## Exported units

- **the channel itself** — holds the bound constant handle and a reference to the submission context it writes through.
- **`set_LOD(handle)`** — bind: remember which constant to write.
- **`set_LOD(real)`** — push: map the scalar to a subdivision factor and write it.
- **`unmap`** — forget the binding; called when the program changes and at construction.
