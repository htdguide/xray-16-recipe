# src/Layers/xrRenderDX11/3DFluid/dx113DFluidBlenders.h

> Declares the eight pass families the fluid subsystem builds for itself rather than loading from game data.

**Needs** — [`dx113DFluidBlenders.cpp`](dx113DFluidBlenders.cpp.md) · [`xrRender/Blender.h`](../../xrRender/Blender.h.md)
**Used by** — [`dx113DFluidBlenders.cpp`](dx113DFluidBlenders.cpp.md) · [`dx113DFluidEmitters.cpp`](dx113DFluidEmitters.cpp.md) · [`dx113DFluidManager.cpp`](dx113DFluidManager.cpp.md) · [`dx113DFluidObstacles.cpp`](dx113DFluidObstacles.cpp.md) · [`dx113DFluidRenderer.cpp`](dx113DFluidRenderer.cpp.md)
**Tier floor** — T2: it is a list of pass-family declarations; the device facts are in the implementation.

## Purpose

Declares the surface implemented in [`dx113DFluidBlenders.cpp`](dx113DFluidBlenders.cpp.md).

## Exported units

Eight pass families, each compiling a numbered set of related passes:

- **`CBlender_fluid_advect`** — advection of a scalar field, plain or error-compensated, for density or temperature.
- **`CBlender_fluid_advect_velocity`** — advection of the velocity field, with or without buoyancy.
- **`CBlender_fluid_simulate`** — vorticity, confinement, divergence, pressure relaxation, projection.
- **`CBlender_fluid_obst`** — rasterizing a static or a moving oriented box into the occupancy field.
- **`CBlender_fluid_emitter`** — depositing a gaussian blob into a field.
- **`CBlender_fluid_obstdraw`** — the flat-atlas diagnostic view of a field.
- **`CBlender_fluid_raydata`** — computing each view ray's entry point and length inside the volume.
- **`CBlender_fluid_raycast`** — edge detection, the march, and the composite to the frame.

**Notes** — Every one of them declares itself ineligible for the two material features the engine otherwise offers — detail texturing and lightmapping — and carries a comment marking it internal. Those declarations exist because the material system asks every blender whether it supports them; the honest answer for a simulation pass is no. A rebuild whose materials are not all the same kind of thing will not need the question.
