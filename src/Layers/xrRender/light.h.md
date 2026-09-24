# src/Layers/xrRender/light.h

> Declares the light source: the chapter-4 light interface joined to a spatial-database entry.

**Needs** — [`light.cpp`](light.cpp.md) · [`xrCDB/ISpatial.h`](../../xrCDB/ISpatial.h.md) · [`Light_Package.h`](Light_Package.h.md) · [`light_smapvis.h`](light_smapvis.h.md) · [`light_gi.h`](light_gi.h.md) · [`r_sun_cascades.h`](r_sun_cascades.h.md) · [`Shader.h`](Shader.h.md)
**Used by** — [`LightTrack.cpp`](LightTrack.cpp.md) · [`LightTrack.h`](LightTrack.h.md) · [`Light_DB.cpp`](Light_DB.cpp.md) · [`Light_DB.h`](Light_DB.h.md) · [`Light_Package.cpp`](Light_Package.cpp.md) · [`Light_Package.h`](Light_Package.h.md) · [`Light_Render_Direct.h`](Light_Render_Direct.h.md) · [`Light_Render_Direct_ComputeXFS.cpp`](Light_Render_Direct_ComputeXFS.cpp.md) · [`light.cpp`](light.cpp.md) · [`light_gi.cpp`](light_gi.cpp.md) · [`light_smapvis.cpp`](light_smapvis.cpp.md) · [`light_vis.cpp`](light_vis.cpp.md) · [`r__dsgraph_build.cpp`](r__dsgraph_build.cpp.md) · [`r__pixel_calculator.cpp`](r__pixel_calculator.cpp.md) · _and 8 more_
**Tier floor** — T1: it fixes the light's memory layout, including a bit-packed flag word and a union over the three projection shapes, because a level's lights are walked as an array every frame.

## Purpose

Declares the type implemented in [`light.cpp`](light.cpp.md), [`light_vis.cpp`](light_vis.cpp.md) and [`light_gi.cpp`](light_gi.cpp.md). Its one structural decision is worth stating here because it shapes every user: a light is *simultaneously* the renderer's light object and an entry in the spatial database, by inheritance in the original. It is indexed, queried and culled by the same machinery as renderable objects and sound sources, and it answers "which sector am I in" the same way they do.

Exported units:

- **`light`** — the light source. Its state, invariants and algorithms are described in [`light.cpp`](light.cpp.md); its occlusion testing in [`light_vis.cpp`](light_vis.cpp.md); its indirect-bounce generation in [`light_gi.cpp`](light_gi.cpp.md).

**Notes** — Two shapes in the declaration are decisions a rebuild must reckon with rather than copy:

- The three projection forms — directional (one per sun cascade), point and spot — occupy the *same storage*, because a light is exactly one type for its whole life and the directional form is large. A rebuild with a tagged union gets the same effect; one with a class per type pays an indirection in the deferred path's hottest loop.
- The whole deferred-path half of the record — attenuation, cone children, indirect bounces, shadow-map visibility, materials, projections — is absent from the oldest renderer, which needs only a position, colour and range. A rebuild that supports only the deferred path deletes the split and keeps the second half.
