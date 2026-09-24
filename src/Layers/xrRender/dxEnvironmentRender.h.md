# src/Layers/xrRender/dxEnvironmentRender.h

> Declares the renderer-side filling of the sky and weather port.

**Needs** — [`Include/xrRender/EnvironmentRender.h`](../../Include/xrRender/EnvironmentRender.h.md) · [`dxEnvironmentRender.cpp`](dxEnvironmentRender.cpp.md)
**Used by** — [`dxEnvironmentRender.cpp`](dxEnvironmentRender.cpp.md) · [`r4_rendertarget_phase_combine.cpp`](../xrRenderPC_R4/r4_rendertarget_phase_combine.cpp.md)
**Tier floor** — T2: a declaration only.

## Purpose

Declares the surface implemented in [`dxEnvironmentRender.cpp`](dxEnvironmentRender.cpp.md).

Exported units:

- **`dxEnvDescriptorRender`** — the per-time-of-day texture triple (sky cube, environment cube, cloud layer), created and destroyed with the device and copied wholesale when the weather system clones a descriptor.
- **`dxEnvironmentRender`** — the sky and cloud renderer: the two materials and vertex formats, the two texture lists rebuilt each frame, the five named placeholder textures the rest of the engine reaches the sky through, and the resolved sampler stage indices. Implements sky and cloud drawing, device create/destroy, the per-frame descriptor blend, and access to the particle library.
