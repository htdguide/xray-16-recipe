# src/Layers/xrRender/FBasicVisual.h

> Declares what every drawable model is: a type tag, a bounding volume, one material, and a draw call — plus the mesh record that binds a slice of the shared geometry buffers.

**Needs** — [`Include/xrRender/RenderVisual.h`](../../Include/xrRender/RenderVisual.h.md) · [`xrEngine/vis_common.h`](../../xrEngine/vis_common.h.md) · [`BufferUtils.h`](BufferUtils.h.md) · [`Shader.h`](Shader.h.md)
**Used by** — [`FBasicVisual.cpp`](FBasicVisual.cpp.md) · [`FHierrarhyVisual.h`](FHierrarhyVisual.h.md) · [`FTreeVisual.h`](FTreeVisual.h.md) · [`FVisual.h`](FVisual.h.md) · [`ModelPool.cpp`](ModelPool.cpp.md) · [`ModelPool.h`](ModelPool.h.md) · [`ParticleEffect.cpp`](ParticleEffect.cpp.md) · [`ParticleEffect.h`](ParticleEffect.h.md) · [`dxParticleCustom.cpp`](dxParticleCustom.cpp.md) · [`dxParticleCustom.h`](dxParticleCustom.h.md) · [`light_smapvis.cpp`](light_smapvis.cpp.md) · [`light_smapvis.h`](light_smapvis.h.md) · [`r__dsgraph_build.cpp`](r__dsgraph_build.cpp.md) · [`r__dsgraph_render.cpp`](r__dsgraph_render.cpp.md) · _and 5 more_
**Tier floor** — T1: the mesh record holds device buffer references with explicit base offsets and counts.

## Purpose

Declares the surface implemented in [`FBasicVisual.cpp`](FBasicVisual.cpp.md), and with it the **root of the whole model type hierarchy**. Everything the renderer can draw as an object — a wall, a crate, a creature, a tree, a particle emitter, a level-of-detail impostor — derives from the one type declared here. The chapter README lists the derived types.

## Exported units

- **`RenderMesh`** — one drawable slice of geometry: a geometry declaration, a vertex buffer with a base index and a count, an index buffer with a base and a count, and the resulting primitive count. It is a *view* into buffers it shares with other meshes; it does not own them exclusively.
- **`Visual`** — the model base: its type tag, its visibility record (bounding box and sphere, plus the occlusion bookkeeping the renderer writes into), its material, and the four operations every model supports — load from a chunked stream, draw at a given level of detail, copy from another, and release.
- **`VLOAD_NOVERTICES`** — a load flag meaning "read the description but do not build geometry", used when only the model's bounds and structure are wanted.

**Notes** — `Visual` declares `spawn` and `depart` as empty hooks. They are the instance-lifetime notifications the particle and animated types need and nothing else uses; they belong on those types, not here. Their presence on the base is an artefact of the base being the only thing the model pool knows about.

The mesh record is deliberately non-copyable in the original, because copying one would duplicate a buffer reference without adjusting its reference count. That is a C++ hazard; the decision it protects is that **a mesh does not own its buffers**, which a rebuild expresses in its own ownership vocabulary.
