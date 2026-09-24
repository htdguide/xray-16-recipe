# src/Layers/xrRender/dxObjectSpaceRender.h

> Declares the renderer-side filling of the collision debug-view port.

**Needs** — [`Include/xrRender/ObjectSpaceRender.h`](../../Include/xrRender/ObjectSpaceRender.h.md) · [`xrEngine/xr_collide_form.h`](../../xrEngine/xr_collide_form.h.md) · [`dxObjectSpaceRender.cpp`](dxObjectSpaceRender.cpp.md)
**Used by** — [`dxObjectSpaceRender.cpp`](dxObjectSpaceRender.cpp.md)
**Tier floor** — T2: a declaration only.

## Purpose

Declares the surface implemented in [`dxObjectSpaceRender.cpp`](dxObjectSpaceRender.cpp.md): the concrete collision debug view that satisfies [`IObjectSpaceRender`](../../Include/xrRender/ObjectSpaceRender.h.md). The whole declaration exists only in debug builds.

Exported units:

- **`dxObjectSpaceRender`** — holds the wireframe material, the collision query record the collision system writes its tested boxes into, and the queue of pending spheres. Implements the draw-and-clear, the two queueing calls and the bare material bind. Both list members are reachable from more than one thread and the declaration says so.
