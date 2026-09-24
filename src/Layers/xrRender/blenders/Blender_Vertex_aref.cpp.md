# src/Layers/xrRender/blenders/Blender_Vertex_aref.cpp

> The vertex-lit world surface with alpha testing: the same template, for cut-out geometry such as grates, chain-link and foliage cards.

**Needs** — [`Blender_Vertex_aref.h`](Blender_Vertex_aref.h.md) · [`Blender.h`](../Blender.h.md) · [`Blender_Recorder.h`](../Blender_Recorder.h.md) · [`Blender_CLSID.h`](../Blender_CLSID.h.md) · [`Shader.h`](../Shader.h.md) · [`xrRender_console.h`](../xrRender_console.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: it emits a pass description; the parameter block is a frozen tagged byte stream.

## Purpose

Identical in structure to the plain vertex-lit template, with two parameters added: an alpha reference and a flag choosing between *cutting out* and *blending*. The distinction matters because it decides whether the surface can be drawn in the opaque pass at all.

Forward renderer only.

## State

```text
RECORD Parameters
  alpha_ref   : int in [0,255], default 32   # texel alpha below this is discarded
  alpha_blend : bool, default false          # blend instead of merely cutting out
```

**Invariants** — alpha testing is **always on** for this template, whichever way the flag goes; the flag only selects the blend factors. That is why the alpha reference is meaningful even in the non-blending case: a cut-out surface is opaque everywhere it survives the test.

Unlike the plain vertex template, this one answers **no** to "can be detailed" — it inherits the base's default — yet its programmable path binds a detail texture whenever the compiler resolved one. That is unreachable in practice, because the compiler only resolves a detail texture for a template that said yes, so the detail branches here are dead. A rebuild should either add the capability or drop the branches; the shipped art works either way.

## `Save` / `Load`

**Contract** — writes the alpha reference then the blend flag. Version 0 has only the reference and defaults the flag to false.

## `Compile` — the fixed-function path

```text
FUNCTION compile_fixed(context)
  IF lightmaps and dynamic lights are both off THEN
      one pass, lighting and fog on, one stage: base texture * vertex colour
      # NOTE: no alpha test is set here at all — the fallback shape draws
      # the cut-out surface fully opaque. See Notes.
      RETURN

  SELECT context.element
    normal_hq, normal_lq ->
      depth test and write on
      blend := alpha_blend ? (src alpha : inverse src alpha) : (one : zero)
      alpha test on at alpha_ref
      fog on, fixed lighting off
      one stage: base texture DOUBLED against vertex colour
    lighting_only ->
      same blend and alpha test, no lighting and no fog
      one stage: colour from vertex colour, alpha from the texture
```

**Notes** — The blending pair is set with *blending enabled* even in the non-blending case, where the factors are one and zero. The material compiler normalizes that back to disabled when it records the state, so the two spellings intern to the same pipeline state; writing it this way keeps the alpha-test argument in one call.

The fallback shape's missing alpha test is a defect, not a decision — the line that would set it is present but commented out in the source. A rebuild should set it: without it, foliage in that configuration draws as opaque rectangles.

## `Compile` — the programmable path

**Contract** — one pass per element. Every element carries the alpha test; the reference is the parameter for the world passes and the parameter again for the light passes.

```text
FUNCTION compile_programmable(context)
  blend := alpha_blend ? (src alpha : inverse src alpha) : (one : zero)

  SELECT context.element
    normal_hq -> programs (detail_diffuse ? "vert_dt" : "vert")
                 fog on, blend as above, alpha test at alpha_ref
                 bind s_base; if detailed, s_detail
    normal_lq -> programs "vert", same state, bind s_base
    add_point -> programs "vert_point" / "add_point"
                 depth write off, blend one:one, alpha test at alpha_ref
                 bind s_base, s_lmap and s_att <- point attenuation
    add_spot  -> programs "vert_spot" / "add_spot"
                 depth write off, blend one:one, alpha test at alpha_ref
                 bind s_base, s_lmap <- spot cookie with projective division,
                 s_att <- spot attenuation
    lighting_only -> programs "vert_l", bind s_base
```

**Invariants** — the additive light passes keep the same alpha reference as the base pass. They must: a light pass that admitted texels the base pass discarded would light holes in the geometry.
