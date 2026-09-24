# src/Layers/xrRender/TextureDescrManager.h

> Declares the texture-description database's surface.

**Needs** — [`ETextureParams.h`](ETextureParams.h.md) · [`r_constants.h`](r_constants.h.md)
**Used by** — [`Blender_Recorder.cpp`](Blender_Recorder.cpp.md) · [`ResourceManager.cpp`](ResourceManager.cpp.md) · [`ResourceManager.h`](ResourceManager.h.md) · [`ResourceManager_Loader.cpp`](ResourceManager_Loader.cpp.md) · [`ResourceManager_Scripting.cpp`](ResourceManager_Scripting.cpp.md) · [`TextureDescrManager.cpp`](TextureDescrManager.cpp.md) · [`uber_deffer.cpp`](blenders/uber_deffer.cpp.md)
**Tier floor** — T2: a declaration only.

## Purpose

Declares the surface implemented in [`TextureDescrManager.cpp`](TextureDescrManager.cpp.md), plus the three private record shapes that make up one texture's description.

## Exported units

- **`CTextureDescrMngr`** — the database. `Load` fills it from the four sources in precedence order; `UnLoad` empties the description table while keeping the detail scalers alive.
- **`GetBumpName`** — the paired normal map's asset name, or empty.
- **`GetMaterial`** — the surface-material coordinate, defaulting to 1.0.
- **`GetTextureUsage`** — how the detail layer applies (diffuse, bump); leaves its outputs untouched when the texture is undescribed.
- **`GetDetailTexture`** — the detail texture's name and its per-frame constant binder, if any.
- **`UseSteepParallax`** — whether the bump map carries a height channel.
