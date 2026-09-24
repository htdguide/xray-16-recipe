# src/Layers/xrRender/blenders/Blender_default_aref.cpp

> The lightmapped world surface with alpha testing: the workhorse template, for cut-out geometry that still takes a baked lightmap.

**Needs** — [`Blender_default_aref.h`](Blender_default_aref.h.md) · [`Blender.h`](../Blender.h.md) · [`Blender_Recorder.h`](../Blender_Recorder.h.md) · [`Blender_CLSID.h`](../Blender_CLSID.h.md) · [`Shader.h`](../Shader.h.md) · [`xrRender_console.h`](../xrRender_console.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: it emits a pass description; the parameter block is a frozen tagged byte stream.

## Purpose

The lightmapped-diffuse template with an alpha test bolted on. Used wherever a large, lightmapped surface has holes in it: fences, ladders, corrugated roofing, the big foliage cards. Detailable and lightmappable, exactly like the plain template.

Forward renderer only; the deferred renderers answer the same class tag with [`blender_deffer_aref.cpp`](blender_deffer_aref.cpp.md).

## State

```text
RECORD Parameters
  alpha_ref   : int in [0,255], default 32
  alpha_blend : bool, default false
```

## `Save` / `Load`

**Contract** — alpha reference then blend flag. Version 0 carries only the reference and defaults the flag false.

## `Compile` — the fixed-function path

```text
FUNCTION compile_fixed(context)
  IF lightmaps and dynamic lights are both off THEN
      one pass:
        blend := alpha_blend ? (src alpha : inverse src alpha) : (one : zero)
        alpha test on at alpha_ref, lighting and fog on
        stage 0: base texture * vertex colour, wrap addressing
      RETURN

  SELECT context.element
    normal_hq, normal_lq ->
      depth test and write on, blend as above, alpha test at alpha_ref, fog on
      stage 0: the lightmap, if lightmaps are on
      stage 1: base texture DOUBLED against the running colour;
               ALPHA taken from the base texture alone
    lighting_only ->
      depth test and write on, replace, fog on, NO alpha test
      stage 0: the lightmap only
```

**Invariants** — the base stage takes its alpha from the texture and not from the running value. It has to: the running value at that point carries the lightmap's alpha, and testing against that would cut out by illumination rather than by the texture's own mask.

**Notes** — The lighting-only element drops the alpha test. That element exists to feed the lightmap-capture and model-shadow paths, which want the surface's full silhouette, not its cut-out one — a shadow cast through a chain-link fence at this resolution is noise.

## `Compile` — the programmable path

**Contract** — one pass per element; requires the instance to name at least three textures (base, lightmap, hemispheric ambient). Fewer is fatal.

```text
FUNCTION compile_programmable(context)
  blend := alpha_blend ? (src alpha : inverse src alpha) : (one : zero)

  SELECT context.element
    normal_hq -> programs (detail_diffuse ? "lmap_dt" : "lmap")
                 fog on, depth test and write on, blend as above, alpha test at alpha_ref
                 bind s_base, s_lmap, s_hemi (through the render-target sampler)
                 IF detail_diffuse THEN bind s_detail
    normal_lq -> programs "lmap", same state, bind s_base, s_lmap, s_hemi

    add_point -> IF alpha_blend THEN emit nothing
                 programs "lmap_point" / "add_point"
                 depth write off, blend one:one, alpha test at alpha_ref
                 bind s_base, s_lmap and s_att <- point attenuation
    add_spot  -> IF alpha_blend THEN emit nothing
                 programs "lmap_spot" / "add_spot"
                 depth write off, blend one:one, alpha test at alpha_ref
                 bind s_base, s_lmap <- spot cookie with projective division,
                 s_att <- spot attenuation

    lighting_only -> programs "lmap_l", bind s_base, s_lmap, s_hemi
```

**Invariants** — **a blended surface receives no dynamic lights.** Both additive light elements emit an empty pass list when the blend flag is set, and that is the correct decision rather than an omission: an additive pass over a surface that is itself blended into the frame double-counts the background, so the light would brighten whatever is behind the glass rather than the glass. The consequence a rebuilder must accept is that blended surfaces in the forward renderer are lit only by the bake.
