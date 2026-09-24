# src/Layers/xrRender/blenders/Blender_BmmD.cpp

> The terrain template: a lightmapped base texture with a second, tiled "implicit detail" texture named by the material itself rather than resolved from the texture database.

**Needs** — [`Blender_BmmD.h`](Blender_BmmD.h.md) · [`Blender.h`](../Blender.h.md) · [`Blender_Recorder.h`](../Blender_Recorder.h.md) · [`Blender_CLSID.h`](../Blender_CLSID.h.md) · [`Shader.h`](../Shader.h.md) · [`xrRender_console.h`](../xrRender_console.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: it emits a pass description; the parameter block is a frozen tagged byte stream.

## Purpose

"Implicit times detail": the ground. A large, low-resolution base texture supplies the colour variation across a whole terrain sector; a second texture, named by the template, supplies the close-up grain and is tiled far more densely. Unlike the ordinary detail layer, this second texture is **the material's own parameter**, so terrain can carry a detail layer chosen per material rather than per base texture.

The template also carries four more texture names — one per channel of a *mask* texture — which the forward renderer never uses and the deferred renderers do. They are described here because the parameter block is shared and frozen; their meaning is in [`Blender_BmmD_deferred.cpp`](Blender_BmmD_deferred.cpp.md).

Forward renderer filling.

## State

```text
RECORD Parameters
  detail_texture : text, default "$null"   # the tiled second layer
  detail_matrix  : text, default "$null"   # its uv transform
  channel_r      : text, default "detail/detail_grnd_grass"
  channel_g      : text, default "detail/detail_grnd_asphalt"
  channel_b      : text, default "detail/detail_grnd_earth"
  channel_a      : text, default "detail/detail_grnd_yantar"
```

**Invariants** — the four channel defaults are **real shipped asset names**, not placeholders. A material authored before the four-channel feature existed (parameter version below 3) gets them, and they must be these exact four, in this order, or a terrain material saved by an old tool renders with the wrong ground types.

## `Save` / `Load`

**Contract** — a marker, then the detail texture and matrix, then the four channel textures. Reading is version-gated: below version 3, only the marker, texture and matrix are present and the four channels keep their defaults; at version 3 all six are read.

## `Compile` — the fixed-function path

```text
FUNCTION compile_fixed(context)
  IF lightmaps and dynamic lights are both off THEN
      depth test and write on, replace, lighting and fog on
      stage 0: base texture * vertex colour
      stage 1: detail texture, DOUBLED against the running colour;
               alpha taken from the running value
  ELSE
      depth test and write on, replace, fog on, fixed lighting off
      stage 0: base texture selected (not modulated — the lightmap
               arrives through the programmable path or not at all)
      stage 1: detail texture, doubled against the running colour
```

**Notes** — Both shapes ignore the element entirely: the terrain template emits the same single pass whatever is being drawn. It also never samples a lightmap on this path, despite answering yes to the lightmap capability. The recoverable reading is that terrain's lightmap arrives as vertex colour in this configuration; what the doubled second stage is compensating for is the same half-intensity convention as everywhere else.

## `Compile` — the programmable path

**Contract** — one pass per element, requiring at least two instance textures (base and lightmap). Fewer is fatal, and the failure names the base texture so the offending material can be found.

```text
FUNCTION compile_programmable(context)
  SELECT context.element
    normal_hq, normal_lq ->                 # identical; terrain has no quality split
      programs "impl_dt", with fog
      bind s_base   <- instance texture 0
      bind s_lmap   <- instance texture 1
      bind s_detail <- THE TEMPLATE'S detail texture, not a resolved one
    add_point ->
      programs "impl_point" / "add_point"
      depth write off, blend one:one, alpha test at 0
      bind s_base, s_lmap and s_att <- point attenuation
    add_spot ->
      programs "impl_spot" / "add_spot"
      depth write off, blend one:one, alpha test at 0
      bind s_base, s_lmap <- spot cookie with projective division,
      s_att <- spot attenuation
    lighting_only ->
      programs "impl_l", bind s_base and s_lmap
```

**Invariants** — `s_detail` is bound from the template's parameter here, where every other template binds it from the material compiler's resolved detail texture. That is the whole point of the template and it is why terrain materials do not need an entry in the texture-description database.
