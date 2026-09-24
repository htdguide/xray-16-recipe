# src/Layers/xrRender_R2/r2_rendertarget_draw_volume.cpp

> Draws the bounding mesh that belongs to a light's type.

**Needs** — [`r2_rendertarget.cpp`](r2_rendertarget.cpp.md) · [`xrRender/light.h`](../xrRender/light.h.md)
**Used by** — [`r2_rendertarget_accum_reflected.cpp`](r2_rendertarget_accum_reflected.cpp.md) · [`r3_rendertarget_accum_point.cpp`](r3_rendertarget_accum_point.cpp.md) · [`r3_rendertarget_accum_spot.cpp`](r3_rendertarget_accum_spot.cpp.md)
**Tier floor** — T2: a dispatch on an enumeration.

## Purpose

Every step of light accumulation — marking the stencil with back faces, clearing it with
front faces, running the lighting program, and the blend-copy tail — draws the *same*
mesh with different state. Factoring the draw out is what lets those steps read as state
changes rather than as repeated geometry setup, and it is the single place that knows
which mesh a light type uses.

## `draw_volume`

**Contract** — binds the mesh for this light's type and issues one indexed draw of the
whole mesh. The caller has already set the world transform to the light's, and the view
and projection to the camera's. A light type with no volume draws nothing.

```text
FUNCTION draw_volume(light)
  SELECT light.type
    point, reflected -> the sphere
    spot             -> the cone
    omni-part        -> the sphere cap
    otherwise        -> nothing
```

**Notes** — a reflected (indirect-bounce) light uses the sphere even though it has a
direction, because its cone is a full hemisphere; the directionality lives entirely in the
lighting program's weighting, not in the volume. See
[`r2_R_lights.cpp`](r2_R_lights.cpp.md) for where those lights come from.
