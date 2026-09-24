# src/xrPhysics/params.h

> Declares the one physics tuning value that comes from configuration rather than from a console variable.

**Needs** — [`params.cpp`](params.cpp.md)
**Used by** — [`PHWorld.cpp`](PHWorld.cpp.md) · [`params.cpp`](params.cpp.md)
**Tier floor** — T3: a single scalar and a loader.

## Purpose

Declares the surface implemented in [`params.cpp`](params.cpp.md).

- `object_damage_factor` — the global scale on damage dealt by physical objects striking creatures.
- `load_params` — reads it from the configuration.
