# src/Layers/xrRender/blenders/blender_light_direct.h

> Declares the sun-accumulation templates.

**Needs** — [`Blender.h`](../Blender.h.md) · [`blender_light_direct.cpp`](blender_light_direct.cpp.md)
**Used by** — [`blender_light_direct.cpp`](blender_light_direct.cpp.md)
**Tier floor** — T2: a declaration only.

## Purpose

Declares the surface implemented in [`blender_light_direct.cpp`](blender_light_direct.cpp.md). None of these templates is named by the shipped data: they carry no class identifier and the renderer instantiates them directly, which is what "INTERNAL" in their description means.

Exported units:

- **`CBlender_accum_direct`** — the sun accumulation template: one pass per shadow cascade plus a luminance pass.
- **`CBlender_accum_direct_msaa`** — the same, compiled for one multisample sample index carried as a name/definition pair.
- **`CBlender_accum_direct_volumetric_msaa`** — in-scattering inside the light's own volume.
- **`CBlender_accum_direct_volumetric_sun_msaa`** — screen-wide in-scattering.

The three sample-indexed variants exist only on the backends that do per-sample shading; the plain one exists everywhere. All four answer *no* to both "can be detailed" and "can be lightmapped" — they are screen-space effects, not surfaces.
