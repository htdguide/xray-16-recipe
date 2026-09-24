# src/Layers/xrRender/blenders/Blender_detail_still.cpp

> The grass-and-debris template: the detail-object layer, drawn with a vertex program that either sways it in the wind or holds it still.

**Needs** — [`Blender_detail_still.h`](Blender_detail_still.h.md) · [`Blender.h`](../Blender.h.md) · [`Blender_Recorder.h`](../Blender_Recorder.h.md) · [`Blender_CLSID.h`](../Blender_CLSID.h.md) · [`Shader.h`](../Shader.h.md) · [`xrRender_console.h`](../xrRender_console.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: it emits a pass description; the parameter block is a frozen tagged byte stream.

## Purpose

**Detail objects** are the grass, weeds and small rubble scattered over a level's ground by the detail-object layer. They are drawn as one enormous batch of tiny meshes, and this template is what that batch is drawn with.

The interesting decision is how the wind gets in. There is no wind parameter and no per-object animation: the *shader element* selects the vertex program, and the high-quality element names the swaying one while the low-quality element names the still one. Quality setting and wind animation are therefore the same switch — turning detail quality down freezes the grass. That is the engine's decision, and it is why the class is named "still" while its high-quality element waves.

Forward renderer filling; the deferred renderers use [`Blender_detail_still_deferred.cpp`](Blender_detail_still_deferred.cpp.md).

## State

```text
RECORD Parameters
  alpha_blend : bool, default false
```

## `Save` / `Load`

**Contract** — one boolean, unversioned.

## `Compile` — the fixed-function path

```text
FUNCTION compile_fixed(context)
  pass:
    depth test and write on
    blend := alpha_blend ? blend-by-alpha with a test at 200
                         : replace       with a test at 200

    IF lightmaps and dynamic lights are both off THEN
        lighting and fog on
        stage 0: base texture * vertex colour
    ELSE
        lighting and fog OFF
        SELECT context.element
          normal_hq -> vertex program "detail_wave", pixel program none
                       stage 0: base texture DOUBLED against vertex colour
          normal_lq -> vertex program "detail_still", pixel program none
                       stage 0: base texture DOUBLED against vertex colour
          lighting_only -> vertex program "detail_still", pixel program none
                       stage 0: vertex colour, alpha from the texture
```

**Invariants** — the alpha test is at **200** in every shape, blended or not, and there is no parameter for it. Grass cards are cut-outs with hard edges; a low reference leaves a halo of semi-transparent texels that reads as fog at the blade edges. 200 is high enough to cut cleanly and low enough to keep the blade's full silhouette.

**Notes** — Even the fixed-function path names a vertex program while leaving the pixel side to the fixed combine chain. That mixed form is legal — the wind displacement is per-vertex and has no pixel-side counterpart — and it is the only template here that uses it.

## `Compile` — the programmable path

```text
FUNCTION compile_programmable(context)
  SELECT context.element
    normal_hq -> programs "detail_wave" / "detail"
    normal_lq -> programs "detail_still" / "detail"
    anything else -> emit nothing
  both: fog off, depth test and write on, blend replace,
        alpha test on at 200 only when alpha_blend is set
  bind s_base <- instance texture 0
```

**Invariants** — detail objects receive **no dynamic lights** in the forward renderer: the point and spot elements emit nothing. The batch is far too large to draw once per light, and the layer's lighting comes entirely from the vertex colours the detail-object layer bakes.
