# src/Layers/xrRender/blenders/dx11RainBlender.h

> Declares the wet-surface templates.

**Needs** — [`Blender.h`](../Blender.h.md) · [`dx11RainBlender.cpp`](dx11RainBlender.cpp.md)
**Used by** — [`dx11RainBlender.cpp`](dx11RainBlender.cpp.md) · [`glRainBlender.cpp`](glRainBlender.cpp.md)
**Tier floor** — T2: a declaration only.

## Purpose

Declares the surface implemented in [`dx11RainBlender.cpp`](dx11RainBlender.cpp.md), and — because the classes are shared — in [`glRainBlender.cpp`](glRainBlender.cpp.md). Exactly one of those two files is built.

Exported units:

- **`CBlender_rain`** — internal template, no class identifier, neither detailable nor lightmappable; emits the wet-surface passes.
- **`CBlender_rain_msaa`** — the same, compiled for one multisample sample index carried as a name/definition pair.
