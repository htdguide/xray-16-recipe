# src/xrEngine/Engine.h

> Declares the engine's composition root, and the render-context count that fixes how much of a frame can be recorded in parallel.

**Needs** — [`pure.h`](pure.h.md) · [`EngineAPI.h`](EngineAPI.h.md) · [`EventAPI.h`](EventAPI.h.md) · [`xrSheduler.h`](xrSheduler.h.md) · [`xrSound/Sound.h`](../xrSound/Sound.h.md)
**Used by** — [`pch.hpp`](../editors/xrWeatherEditor/pch.hpp.md) · [`Engine.cpp`](Engine.cpp.md) · [`EngineAPI.h`](EngineAPI.h.md) · [`ISheduled.cpp`](ISheduled.cpp.md) · [`ISheduled.h`](ISheduled.h.md) · [`Stats.cpp`](Stats.cpp.md) · [`profiler.h`](profiler.h.md) · [`stdafx.h`](stdafx.h.md) · [`x_ray.cpp`](x_ray.cpp.md) · [`x_ray.h`](x_ray.h.md) · [`xr_ioc_cmd.cpp`](xr_ioc_cmd.cpp.md) · [`stdafx.h`](../xr_3da/stdafx.h.md)
**Tier floor** — T2.

## Purpose

Declares the surface implemented in [`Engine.cpp`](Engine.cpp.md), and defines the render-context budget that the renderer and everything that submits to it are sized against.

## State

```text
# Render context budget — a compile-time constant set, not configuration.
sun_cascade_contexts   = 3    # one per cascaded shadow-map slice
auxiliary_contexts     = 1    # rain occlusion and spot-light shadow maps share one
parallel_contexts      = 4    # = sun cascades + auxiliary; these record concurrently
total_contexts         = 5    # = parallel + 1 immediate context for the main view
```

Three sun cascades is a quality decision made once and reflected everywhere: the shadow atlas layout, the shader's cascade-select code and the per-cascade distance splits are all built around three. Four parallel contexts plus one immediate is therefore the concurrency the render command recording is designed for, and it is why the visibility pass produces five independent result sets per frame rather than one. The original notes these belong in renderer configuration rather than here; they are in the engine header because both the engine and every backend must agree on the number.

## Exported units

- **`Engine`** — the composition root: module registry, event queue, update scheduler, audio manager. It is itself a frame handler and an event receiver. See [`Engine.cpp`](Engine.cpp.md).
- **`initialize` / `destroy`** — bring up and tear down, in the one order that works.
- **Render-context counts** — as above.

**Notes** — This header also carries the module's symbol-visibility marker, which decides whether the engine is a shared library exporting its surface or is linked into the executable. Which it is is a build option and nothing in the engine's behaviour depends on it; a rebuild whose module system is not the C linker's deletes this entirely.
