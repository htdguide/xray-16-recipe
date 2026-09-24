# src/Layers/xrRender/dxLensFlareRender.h

> Declares the renderer-side filling of the lens-flare port.

**Needs** — [`Include/xrRender/LensFlareRender.h`](../../Include/xrRender/LensFlareRender.h.md) · [`dxLensFlareRender.cpp`](dxLensFlareRender.cpp.md)
**Used by** — [`dxLensFlareRender.cpp`](dxLensFlareRender.cpp.md)
**Tier floor** — T2: a declaration only.

## Purpose

Declares the surface implemented in [`dxLensFlareRender.cpp`](dxLensFlareRender.cpp.md).

Exported units:

- **`dxFlareRender`** — one flare *element*'s material: created from a shader and texture name pair, destroyed on demand. The material is public rather than private so the owning renderer can read it while building the draw list; the encapsulation was given up deliberately and is noted as such in the source.
- **`dxLensFlareRender`** — the flare renderer: holds the vertex format and implements the build-and-draw of the source, the ghost chain and the gradient wash, plus device create and destroy.
