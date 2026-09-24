# src/Layers/xrRender/blenders/Blender_Particle.h

> Declares the particle material template.

**Needs** — [`Blender.h`](../Blender.h.md) · [`Blender_Recorder.h`](../Blender_Recorder.h.md)
**Used by** — [`Blender_Particle.cpp`](Blender_Particle.cpp.md) · [`Blender_Particle_deferred.cpp`](Blender_Particle_deferred.cpp.md)
**Tier floor** — T2: a declaration only.

## Purpose

Declares one template with **two alternative implementations**, exactly one of which is built: [`Blender_Particle.cpp`](Blender_Particle.cpp.md) for the forward renderer and [`Blender_Particle_deferred.cpp`](Blender_Particle_deferred.cpp.md) for the deferred ones.

## Exported units

- **`CBlender_Particle`** — the particle class tag at parameter version 0. Three parameters: a six-way blend-mode selector whose index order is frozen, a texture-clamp flag, and an alpha reference that is stored and never read.
