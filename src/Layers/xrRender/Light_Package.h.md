# src/Layers/xrRender/Light_Package.h

> Declares this frame's visible lights, split into the three buckets the lighting pass draws in.

**Needs** — [`light.h`](light.h.md)
**Used by** — [`Light_DB.cpp`](Light_DB.cpp.md) · [`Light_DB.h`](Light_DB.h.md) · [`Light_Package.cpp`](Light_Package.cpp.md) · [`light.cpp`](light.cpp.md) · [`light.h`](light.h.md)
**Tier floor** — T2: three lists of references.

## Purpose

Declares the surface implemented in [`Light_Package.cpp`](Light_Package.cpp.md).

## Exported units

- **`LightPackage`** — three lists of lights: unshadowed point lights, unshadowed spot lights, and shadowed lights of either kind. Cleared at the start of each frame and refilled by the visibility pass.
- **`clear()` / `sort()`** — empty it, and order each bucket for drawing.

**Notes** — The split into three is the *pass* structure, not a property of lights: unshadowed points and unshadowed spots use different geometry and different shaders, and a shadowed light of any kind needs a shadow map rendered first. A rebuild whose lighting pass is structured differently buckets differently.
