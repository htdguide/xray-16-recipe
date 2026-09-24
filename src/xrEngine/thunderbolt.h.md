# src/xrEngine/thunderbolt.h

> Declares the lightning effect, its authored variants and the palettes weather frames select from; the substance is in [`thunderbolt.cpp`](thunderbolt.cpp.md).

**Needs** — [`thunderbolt.cpp`](thunderbolt.cpp.md) · [`Environment.h`](Environment.h.md) · [`Include/xrRender/ThunderboltRender.h`](../Include/xrRender/ThunderboltRender.h.md) · [`Include/xrRender/ThunderboltDescRender.h`](../Include/xrRender/ThunderboltDescRender.h.md) · [`Include/xrRender/LensFlareRender.h`](../Include/xrRender/LensFlareRender.h.md) · [`Include/xrRender/FactoryPtr.h`](../Include/xrRender/FactoryPtr.h.md)
**Used by** — [`ThunderboltDescRender.h`](../Include/xrRender/ThunderboltDescRender.h.md) · [`ThunderboltRender.h`](../Include/xrRender/ThunderboltRender.h.md) · [`dxThunderboltRender.cpp`](../Layers/xrRender/dxThunderboltRender.cpp.md) · [`editor_environment_thunderbolts_collection.hpp`](../editors/xrWeatherEngine/editor_environment_thunderbolts_collection.hpp.md) · [`editor_environment_thunderbolts_gradient.hpp`](../editors/xrWeatherEngine/editor_environment_thunderbolts_gradient.hpp.md) · [`editor_environment_thunderbolts_manager.hpp`](../editors/xrWeatherEngine/editor_environment_thunderbolts_manager.hpp.md) · [`editor_environment_thunderbolts_thunderbolt.hpp`](../editors/xrWeatherEngine/editor_environment_thunderbolts_thunderbolt.hpp.md) · [`editor_environment_weathers_time.cpp`](../editors/xrWeatherEngine/editor_environment_weathers_time.cpp.md) · [`Environment.cpp`](Environment.cpp.md) · [`Environment_misc.cpp`](Environment_misc.cpp.md) · [`Environment_render.cpp`](Environment_render.cpp.md) · [`thunderbolt.cpp`](thunderbolt.cpp.md)
**Tier floor** — T2.

## Purpose

Declares the surface described in [`thunderbolt.cpp`](thunderbolt.cpp.md).

Exported units:

- **`SThunderboltDesc`** — one authored bolt variant: its geometry, sound, colour animation
  and two glow gradients, all named from one configuration section. Owns its renderer-side
  companion.
- **`SThunderboltDesc::SFlare`** — one glow gradient: opacity, radius pair, and the shader
  and texture names it builds a material from. There are exactly two per variant, at the top
  and centre of the bolt.
- **`SThunderboltCollection`** — a named palette of variants, one per line of its
  configuration section, with a uniform random pick.
- **`CEffect_Thunderbolt`** — the effect: the palettes, the live strike's placement and
  timing, the authored tuning parameters, the per-frame step that writes back into the
  weather frame, and the draw.

## Notes

The header declares a cache size of eight that **nothing in the file or its implementation
reads**. Either a per-strike cache was removed or never written; it is not recoverable and
carries no meaning.

Each renderer backend's thunderbolt filling is granted access to this type's internals by
name. That is the graphics seam leaking: the effect computes a placement and the backend
reaches in for it. A rebuild should pass a render description across the boundary instead —
transform, phase, colour, the two flares — which is exactly the set of fields the backends
actually read.
