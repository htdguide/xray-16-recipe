# src/Layers/xrRenderDX11/3DFluid/dx113DFluidEmitters.h

> Declares what an authored emitter is, and the two passes that feed its contribution into the density and velocity fields.

**Needs** — [`dx113DFluidEmitters.cpp`](dx113DFluidEmitters.cpp.md) · [`dx113DFluidGrid.h`](dx113DFluidGrid.h.md)
**Used by** — [`dx113DFluidData.cpp`](dx113DFluidData.cpp.md) · [`dx113DFluidData.h`](dx113DFluidData.h.md) · [`dx113DFluidEmitters.cpp`](dx113DFluidEmitters.cpp.md) · [`dx113DFluidManager.cpp`](dx113DFluidManager.cpp.md)
**Tier floor** — T2: the emitter record is plain data; its deposit is delegated.

## Purpose

Declares the surface implemented in [`dx113DFluidEmitters.cpp`](dx113DFluidEmitters.cpp.md). The emitter record itself is declared here rather than in [`dx113DFluidData.h`](dx113DFluidData.h.md) even though the volume owns the list, because the parser and the depositor are the only two things that read the fields and this is the file between them.

## Exported units

- **the emitter-type enumeration** — a gaussian blob and a pulsing draught. Only the blob has a pass of its own; the draught is a blob whose speed varies sinusoidally.
- **the emitter record** — position, radius, falloff, flow velocity, density and saturation, the draught's period/phase/amplitude, and the two independent flags selecting whether it feeds the density field, the velocity field, or both.
- **`RenderDensity(volume)`** — deposit every density-contributing emitter.
- **`RenderVelocity(volume)`** — deposit every velocity-contributing emitter.

**Notes** — The draught parameters sit in a union with exactly one member. It is the shape of a variant record for emitter types that were never added, and a rebuild should either make it a real variant or flatten it.
