# src/Layers/xrRender/Light_DB.h

> Declares the level's light set: the static lights baked into the level file, the sun, the separate hemisphere-lighting set, and this frame's visible package.

**Needs** — [`light.h`](light.h.md) · [`Light_Package.h`](Light_Package.h.md)
**Used by** — [`Light_DB.cpp`](Light_DB.cpp.md)
**Tier floor** — T1: the light records are read as byte images out of the level file.

## Purpose

Declares the surface implemented in [`Light_DB.cpp`](Light_DB.cpp.md).

## Exported units

- **`LightDatabase`** — the level's lights. Holds the static set, the hemisphere set, the sun as a distinguished single light, and the per-frame package.
- **`load(reader)`** — read the level's dynamic-light chunk; find the sun in it.
- **`load_hemisphere()`** — read the level compiler's own light file for the ambient-occlusion light set.
- **`create()`** — make a fresh light, defaulted.
- **`add_light(light)`** — submit a light as visible this frame.
- **`update()`** — reposition and recolour the sun from the weather, and clear the package.
- **`sun`** — the one light every renderer generation treats specially.
