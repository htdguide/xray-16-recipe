# src/Layers/xrRenderDX11/3DFluid/dx113DFluidManager.h

> Declares the fluid solver: its shared field slots, its simulation steps, and the one process-wide instance.

**Needs** — [`dx113DFluidManager.cpp`](dx113DFluidManager.cpp.md) · [`dx113DFluidRenderer.h`](dx113DFluidRenderer.h.md)
**Used by** — [`dx113DFluidBlenders.cpp`](dx113DFluidBlenders.cpp.md) · [`dx113DFluidData.cpp`](dx113DFluidData.cpp.md) · [`dx113DFluidManager.cpp`](dx113DFluidManager.cpp.md) · [`dx113DFluidVolume.cpp`](dx113DFluidVolume.cpp.md)
**Tier floor** — T1: it owns device volume textures and their views.

## Purpose

Declares the surface implemented in [`dx113DFluidManager.cpp`](dx113DFluidManager.cpp.md), and two enumerations that are the subsystem's actual structure.

## Exported units

- **the field slot enumeration** — nine volumes in one index space, split by a marker: the slots below the marker are owned by the solver, the ones at and above it are borrowed from whichever fluid volume is being simulated. That marker is the memory-sharing decision made explicit.
- **the simulation-step enumeration** — advect (plain / error-compensated, each in a density and a temperature form), advect velocity (with and without buoyancy), vorticity, confinement, divergence, relaxation, projection. Each names one technique.
- **lifecycle** — `Initialize(width, height, depth)`, `Destroy`, `SetScreenSize` (forwarded to the renderer, whose intermediate targets scale with the window).
- **per-volume operations** — `Update(volume, timestep)`, `RenderFluid(volume)`.
- **queries used by the material scripts** — grid dimensions, the emitter footprint size, and the two name tables: what the engine calls each field, and what the shader sources call it.
- **development hooks** — register, deregister and re-read volume configurations.
- **the process-wide instance**.

**Notes** — The two parallel name tables are the binding mechanism: each field is registered as an engine texture under a user-namespace name, and the material scripts declare a shader resource under the matching plain name. Keeping them as two tables in one file is what stops them drifting apart.
