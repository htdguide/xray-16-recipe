# src/Layers/xrRender/ModelPool.h

> Declares the model cache — base models, cloned instances and the free pool — implemented in [`ModelPool.cpp`](ModelPool.cpp.md).

**Needs** — [`FBasicVisual.h`](FBasicVisual.h.md) · [`ParticleEffectDef.h`](ParticleEffectDef.h.md) · [`ParticleGroup.h`](ParticleGroup.h.md)
**Used by** — [`FHierrarhyVisual.cpp`](FHierrarhyVisual.cpp.md) · [`ModelPool.cpp`](ModelPool.cpp.md) · [`xrRender_console.cpp`](xrRender_console.cpp.md)
**Tier floor** — T2: a declaration only; the T1 pressure lives in the implementation.

## Purpose

Declares the single model cache the renderer owns. The state it describes — the base list, the instance registry and the pool — and every invariant between them are set out in [`ModelPool.cpp`](ModelPool.cpp.md); this page lists only the surface.

The one thing the header decides on its own is that the cache is opened to the scene renderer as a friend, so the renderer can drain the deferred-deletion queue at the frame boundary without that queue being public. A rebuild makes it a method the frame loop calls.

## Exported units

- `Create` — get a visual for a model name, from the pool, from a loaded base, or by loading the file.
- `CreateChild` — get a component visual belonging to another visual; never pooled.
- `CreatePE` · `CreatePG` — build a particle-effect or particle-group visual from an in-memory definition.
- `Delete` — hand a visual back; deferred when a frame is in flight.
- `Discard` — destroy a visual for real and release its base's reference.
- `DeleteInternal` — the undeferred body of `Delete`.
- `DeleteQueue` — drain the deferred deletions between frames.
- `ClearPool` — empty the free pool, optionally releasing the bases too.
- `Prefetch` — warm the cache from the configured per-game-type list.
- `Instance_Create` — empty visual record for a model type identifier.
- `Instance_Load` — read a model file into a visual, from a path or from an open reader.
- `Instance_Duplicate` — clone a base and charge it a reference.
- `Instance_Register` · `Instance_Find` — add and look up a base by name.
- `Logging` — enable or suppress the uncached-load report.
- `dump` · `memory_stats` — diagnostics: per-model memory, and buffer bytes split device/system.
- `Render` · `RenderSingle` · `OnDeviceDestroy` — authoring-tool build only: draw one visual directly through the priority buckets.
