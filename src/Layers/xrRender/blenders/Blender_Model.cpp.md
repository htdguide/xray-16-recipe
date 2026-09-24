# src/Layers/xrRender/blenders/Blender_Model.cpp

> The default template for dynamic models: a base texture lit by the per-object light projector, with optional alpha blending, plus the additive light and shadow-casting elements.

**Needs** — [`Blender_Model.h`](Blender_Model.h.md) · [`Blender.h`](../Blender.h.md) · [`Blender_Recorder.h`](../Blender_Recorder.h.md) · [`Blender_CLSID.h`](../Blender_CLSID.h.md) · [`Shader.h`](../Shader.h.md) · [`xrRender_console.h`](../xrRender_console.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: it emits a pass description; the parameter block is a frozen tagged byte stream.

## Purpose

What every animated or movable object is drawn with unless its material says otherwise — creatures, weapons, crates, doors. A model has no lightmap, so its baked lighting arrives through a **light projector**: a small texture the engine renders per object, holding the aggregate of the static lights reaching it, sampled through a matrix that projects it onto the model. The engine reaches for it by the reserved name `$user$projector`, which is one of the positional-name escapes the material compiler understands.

## State

```text
RECORD Parameters
  alpha_blend  : bool, default false
  alpha_ref    : int in [0,255], default 32
  tessellation : enum { none, pn_triangles, height_map, both }, default none
```

## `Save` / `Load`

**Contract** — writes the blend flag, the alpha reference, then the tessellation selector with its four labels. Reading is version-gated three ways: version 0 has no block and defaults everything; version 1 has the flag and the reference; version 2 adds tessellation.

## `Compile` — the fixed-function path

```text
FUNCTION compile_fixed(context)
  IF lightmaps and dynamic lights are both off THEN
      one pass:
        depth test on; depth WRITE off when blending with a reference below 200
        blend := alpha_blend ? blend-with-alpha-test : replace
        lighting and fog on
        stage 0: base texture * vertex colour, texture alpha
      RETURN

  SELECT context.element
    normal_hq -> depth test and write on, replace, lighting and fog on
                 stage 0: THE LIGHT PROJECTOR, sampled through its own matrix
    normal_lq -> depth test and write on, replace, lighting and fog on
                 stage 0: base texture DOUBLED against vertex colour
```

**Invariants** — depth writing is dropped only for a blended model whose alpha reference is below 200. A high reference means the surface is effectively a cut-out and still lays depth; a low one means it is genuinely translucent and must not.

**Notes** — The high-quality fixed-function element draws the **projector alone** and never samples the base texture. That is visibly wrong and the source carries the two-stage version that samples both, disabled, with a link to the commit that disabled it. The recoverable reading is that the hardware this path targets could not afford the second stage alongside the projector; what cannot be recovered is whether the single-stage result was accepted as correct or simply never looked at. A rebuild targeting only modern hardware should emit the two-stage form.

## `Compile` — the programmable path

**Contract** — one pass per element, naming a program pair per element and folding the blend decision into the pass state.

```text
FUNCTION compile_programmable(context)
  SELECT context.element
    normal_hq -> programs "model_def_hq"
                 IF alpha_blend THEN depth test and write on, blend src-alpha:inv-src-alpha,
                                     alpha test at alpha_ref
                            ELSE plain opaque with fog
                 bind s_base <- instance texture 0
                 bind s_lmap <- the light projector, clamped, projective division on
    normal_lq -> programs "model_def_lq", same blend decision
                 bind s_base only        # no projector at low quality
    add_point -> programs "model_def_point" / "add_point"
                 depth write off, blend one:one, alpha test on
                 (at alpha_ref when blending, otherwise at the default)
                 bind s_base, s_lmap and s_att <- point attenuation
    add_spot  -> programs "model_def_spot" / "add_spot"
                 same state; s_lmap <- spot cookie with projective division,
                 s_att <- spot attenuation
    lighting_only -> programs "model_def_shadow" / "model_shadow"
                     no fog, DEPTH WRITE OFF, blend zero : source colour,
                     no alpha test
                     no textures bound at all
```

**Invariants**

- The light projector is sampled **with projective division** and **clamped**. Clamping is what keeps a model that strays outside its projector's footprint from tiling the projector across itself; the division is because the projector's coordinate arrives in clip space.
- The shadow element multiplies the frame by the source colour and writes no depth. It is not drawing the model — it is *darkening what the model covers*, which is how the forward renderer casts a model's shadow onto the world: a projected darkening pass rather than a shadow map.
