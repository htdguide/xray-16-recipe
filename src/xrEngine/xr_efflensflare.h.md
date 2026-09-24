# src/xrEngine/xr_efflensflare.h

> Declares the lens-flare effect and the authored "sun" descriptions it selects between; the substance is in [`xr_efflensflare.cpp`](xr_efflensflare.cpp.md).

**Needs** — [`xr_efflensflare.cpp`](xr_efflensflare.cpp.md) · [`Environment.h`](Environment.h.md) · [`xrCDB/xr_collide_defs.h`](../xrCDB/xr_collide_defs.h.md) · [`Include/xrRender/LensFlareRender.h`](../Include/xrRender/LensFlareRender.h.md) · [`Include/xrRender/FactoryPtr.h`](../Include/xrRender/FactoryPtr.h.md)
**Used by** — [`dxEnvironmentRender.cpp`](../Layers/xrRender/dxEnvironmentRender.cpp.md) · [`dxLensFlareRender.cpp`](../Layers/xrRender/dxLensFlareRender.cpp.md) · [`editor_environment_manager.cpp`](../editors/xrWeatherEngine/editor_environment_manager.cpp.md) · [`editor_environment_suns_sun.hpp`](../editors/xrWeatherEngine/editor_environment_suns_sun.hpp.md) · [`editor_environment_weathers_time.cpp`](../editors/xrWeatherEngine/editor_environment_weathers_time.cpp.md) · [`Environment.cpp`](Environment.cpp.md) · [`Environment_misc.cpp`](Environment_misc.cpp.md) · [`Environment_render.cpp`](Environment_render.cpp.md) · [`xr_efflensflare.cpp`](xr_efflensflare.cpp.md)
**Tier floor** — T2.

## Purpose

Declares the surface described in [`xr_efflensflare.cpp`](xr_efflensflare.cpp.md).

Exported units:

- **`CLensFlareDescriptor`** — one authored sun: which of the three parts it has, the disc,
  the ghost chain, the gradient, and the rise and fall blend rates. Builds itself from a
  configuration section and owns the materials for every sprite.
- **`CLensFlareDescriptor::SFlare`** — one sprite: opacity, radius, position along the
  flare axis, and its texture and shader names.
- **`CLensFlareDescriptor::SSource`** — the sun disc: a sprite plus a flag that makes it
  ignore the weather's sun colour and draw at full brightness.
- **`CLensFlare`** — the effect. The palette, the cross-fade state machine between two
  descriptions, the smoothed visibility, the camera-relative basis the flare chain is laid
  out along, the per-ray occlusion cache, and the three-way draw.

## Notes

The ray count is fixed at five and exposed as a constant because the cache array is sized
by it. Five is the plus-shaped probe pattern described in the implementation twin; changing
it means choosing a new pattern, not just a new number.

Each renderer backend's lens-flare filling is granted access to this type's internals. As
with the lightning effect, a rebuild should pass an explicit render description across the
graphics seam instead — the backends read the basis vectors, the blend factors, the
gradient value and the current description's sprite lists, and nothing else.
