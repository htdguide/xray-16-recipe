# src/Layers/xrRenderDX11/3DFluid/dx113DFluidData.h

> Declares one fluid volume: its persistent fields, its authored placement and obstacles, its emitter list and its tuning settings.

**Needs** — [`dx113DFluidData.cpp`](dx113DFluidData.cpp.md) · [`dx113DFluidEmitters.h`](dx113DFluidEmitters.h.md)
**Used by** — [`dx113DFluidData.cpp`](dx113DFluidData.cpp.md) · [`dx113DFluidEmitters.cpp`](dx113DFluidEmitters.cpp.md) · [`dx113DFluidManager.cpp`](dx113DFluidManager.cpp.md) · [`dx113DFluidObstacles.cpp`](dx113DFluidObstacles.cpp.md) · [`dx113DFluidRenderer.cpp`](dx113DFluidRenderer.cpp.md) · [`dx113DFluidVolume.cpp`](dx113DFluidVolume.cpp.md) · [`dx113DFluidVolume.h`](dx113DFluidVolume.h.md)
**Tier floor** — T1: it owns volume textures and their targets.

## Purpose

Declares the surface implemented in [`dx113DFluidData.cpp`](dx113DFluidData.cpp.md).

## Exported units

- **the persistent-field enumeration** — velocity, pressure, density. Exactly the state a fluid volume must carry between frames; everything else the solver computes is scratch.
- **the simulation-type enumeration** — fog or fire. Fire advects a temperature field and is rendered with an emissive model; fog advects density and is rendered by absorption.
- **the settings record** — hemisphere light contribution, vorticity confinement scale, density decay, buoyancy, simulation type.
- **`Load`** — read the volume's level record.
- **field accessors** — get and set each field and its target, reference-counted, so the solver can swap a field out.
- **queries** — the placement transform, the obstacle list, the emitter list, the settings.
- **`ReparseProfile`** — development-only configuration reload.
