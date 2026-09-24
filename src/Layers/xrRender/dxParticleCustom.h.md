# src/Layers/xrRender/dxParticleCustom.h

> The renderable form of a particle effect: a visual that is also a particle-system instance.

**Needs** — [`Include/xrRender/ParticleCustom.h`](../../Include/xrRender/ParticleCustom.h.md) · [`FBasicVisual.h`](FBasicVisual.h.md) · [`dxParticleCustom.cpp`](dxParticleCustom.cpp.md)
**Used by** — [`ParticleEffect.cpp`](ParticleEffect.cpp.md) · [`ParticleEffect.h`](ParticleEffect.h.md) · [`ParticleGroup.cpp`](ParticleGroup.cpp.md) · [`ParticleGroup.h`](ParticleGroup.h.md) · [`dxParticleCustom.cpp`](dxParticleCustom.cpp.md)
**Tier floor** — T2: a declaration that joins two surfaces and adds one field.

## Purpose

A particle effect has to be two things at once. To the scene it is a **visual** — something with a bounding volume that the visibility walk can cull and the draw stream can sort. To the game it is a **particle system** — something you start, stop, move and ask whether it has finished. This declaration is the join: it inherits both surfaces and adds the one thing the renderer needs that neither supplies, the vertex format its geometry is drawn with.

It carries no behaviour of its own, and that is the point. Every concrete particle visual in the renderer derives from this and fills in the drawing; the join lives here so that the game side can hold a particle effect without knowing which backend made it.

## Exported units

- **`dxParticleCustom`** — the base of every renderable particle effect. Inherits the renderer's visual base and the particle-effect port, holds the geometry's vertex format, and answers the downcast query that the port uses to ask a visual "are you a particle effect?" — the engine's alternative to a language's own dynamic cast, and the mechanism by which a visual is interrogated for each optional facet it might have. See [`FBasicVisual.h`](FBasicVisual.h.md) for the full facet set.

**Notes** — The destructor is declared and empty. Its only job is to be virtual so that destroying through either base releases the whole object; a rebuild in a language with ordinary single-rooted objects deletes the concern.
