# src/Layers/xrRender/blenders/Blender_Vertex.cpp

> The world-surface template for geometry that carries no lightmap: lighting comes from the vertices themselves.

**Needs** — [`Blender_Vertex.h`](Blender_Vertex.h.md) · [`Blender.h`](../Blender.h.md) · [`Blender_Recorder.h`](../Blender_Recorder.h.md) · [`Blender_CLSID.h`](../Blender_CLSID.h.md) · [`Shader.h`](../Shader.h.md) · [`xrRender_console.h`](../xrRender_console.h.md)
**Used by** — reached through its declarations in [`Blender_Vertex.h`](Blender_Vertex.h.md); callers name that, not this file.
**Tier floor** — T2: it emits a pass description; the parameter block is a frozen tagged byte stream.

## Purpose

The counterpart to the lightmapped template: same job, same detail-texture support, but the baked lighting arrives as per-vertex colour instead of as a second texture. The level compiler assigns this to surfaces too small or too dense to be worth a lightmap chart — trim, pipes, railings, most of the clutter geometry.

This is the forward renderer's filling only; it refuses to build elsewhere.

## State

```text
RECORD Parameters
  tessellation : enum { none, pn_triangles, height_map, both }   # default none
```

**Invariants** — declares itself **detailable** but **not lightmappable**. The second answer is what keeps the level compiler from wasting lightmap area on it.

## `Save` / `Load`

**Contract** — version 0 has no parameter block; version 1 adds the tessellation selector with its four labels written inline. Read is version-gated; a version-0 record leaves the default.

## `Compile` — the fixed-function path

```text
FUNCTION compile_fixed(context)
  IF lightmaps and dynamic lights are both off THEN
      one pass, lighting and fog on, one stage: base texture * vertex colour
      RETURN

  SELECT context.element
    normal_hq, normal_lq ->
      depth test and write on, no blending, fog on, fixed lighting off
      one stage: base texture DOUBLED against vertex colour
    lighting_only ->
      depth test and write on, no blending, no lighting, no fog
      one stage: the vertex colour alone, texture unbound
```

**Notes** — The doubling is the same half-intensity convention the lightmapped template uses, applied to vertex colour instead: the level compiler stores vertex lighting at half range so a surface can exceed its texture's brightness.

## `Compile` — the programmable path

**Contract** — one pass per element. Needs only the base texture from the instance.

```text
FUNCTION compile_programmable(context)
  SELECT context.element
    normal_hq -> programs (detail_diffuse ? "vert_dt" : "vert"), fog on
                 bind s_base <- instance texture 0
                 IF detail_diffuse THEN bind s_detail <- resolved detail texture
    normal_lq -> programs "vert", fog on, bind s_base
    add_point -> programs "vert_point" / "add_point"
                 depth test on, depth write off, blend one:one, alpha test at 0
                 bind s_base, and s_lmap and s_att <- the point attenuation texture
    add_spot  -> programs "vert_spot" / "add_spot"
                 depth test on, depth write off, blend one:one, alpha test at 0
                 bind s_base, s_lmap <- the spot cookie with projective division,
                 s_att <- the spot attenuation texture
    lighting_only -> programs "vert_l", bind s_base
```

**Notes** — The dynamic-light elements bind the attenuation texture to *two* sampler names, `s_lmap` and `s_att`. That is not redundancy: the shader source for the point case reads the same volume texture once as an attenuation term and once in the slot the lightmapped variant uses for its lightmap, so that one pixel program serves both templates. It is why the binding is expressed by name and dropped when unused.
