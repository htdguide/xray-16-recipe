# src/Layers/xrRender/blenders/Blender_Lm(EbB).cpp

> A lightmapped surface whose base texture's alpha channel reveals a reflection underneath — the world-geometry counterpart of the reflective model template.

**Needs** — [`Blender_Lm(EbB).h`](Blender_Lm%28EbB%29.h.md) · [`Blender.h`](../Blender.h.md) · [`Blender_Recorder.h`](../Blender_Recorder.h.md) · [`Blender_CLSID.h`](../Blender_CLSID.h.md) · [`Shader.h`](../Shader.h.md) · [`HWCaps.h`](../HWCaps.h.md) · [`xrRender_console.h`](../xrRender_console.h.md)
**Used by** — reached through its declarations in [`Blender_Lm(EbB).h`](Blender_Lm(EbB).h.md); callers name that, not this file.
**Tier floor** — T2: it emits a pass description; the parameter block is a frozen tagged byte stream.

## Purpose

`lightmap * (environment blended by base)`: the reflective composition, applied to static world geometry that also carries a baked lightmap. Windows, polished floors, water-slick concrete. The blend weight is the base texture's own alpha, exactly as in the reflective model template — that convention is shared and frozen.

Forward renderer filling; used for its own class tag, and the composition is also reached through [`blender_deffer_aref.cpp`](blender_deffer_aref.cpp.md)'s blended path in the deferred renderers.

## State

```text
RECORD Parameters
  env_texture : text, default "$null"
  env_matrix  : text, default "$null"
  alpha_blend : bool, default false     # added at parameter version 1
```

**Invariants** — the parameter version is forced to 1 on every save, so anything this build writes carries the blend flag; a version-0 record leaves it false.

## `Save` / `Load`

**Contract** — a marker, the environment texture, its matrix, and from version 1 the blend flag.

## `Compile` — the fixed-function path

**Contract** — three variants, chosen by the lighting configuration and the device's texture-stage count.

```text
FUNCTION compile_fixed(context)
  IF lightmaps and dynamic lights are both off THEN authoring-tool shape ; RETURN

  SELECT context.element
    normal_hq, normal_lq ->
      IF the device reports exactly 2 stages THEN two-pass variant ELSE three-stage variant
    lighting_only ->
      one pass: the lightmap alone, no fog

# the three-stage variant — one pass
  stage 0: environment map, selected
  stage 1: base texture, BLENDED OVER IT BY THE BASE TEXTURE'S OWN ALPHA;
           alpha passes the base texture through
  stage 2: the lightmap, DOUBLED against the running colour, clamped;
           alpha taken from the running value

# the two-pass variant — the multiply moves into the frame blend
  pass 0: the lightmap alone (depth write on, replace)
  pass 1: environment, then base blended over it by its alpha
          (depth write off, multiply-2x blend into the frame)

# the authoring-tool shape — one pass, vertex colour standing in for the lightmap
  stage 0: environment, clamped, selected
  stage 1: base blended over it by its alpha
  stage 2: vertex colour, modulating
```

**Invariants** — the reflection stage is clamped wherever it is emitted, for the same reason as in the model template: a reflection lookup must saturate, not wrap.

## `Compile` — the programmable path

**Contract** — one pass per element; requires at least two instance textures and samples a third (hemispheric ambient).

```text
FUNCTION compile_programmable(context)
  SELECT context.element
    normal_hq, normal_lq ->
      programs "lmapE"
      IF alpha_blend THEN blend src-alpha:inv-src-alpha with alpha test at 0
                     ELSE opaque with fog
      bind s_base <- instance texture 0
      bind s_lmap <- instance texture 1
      bind s_hemi <- instance texture 2, through the render-target sampler
      bind s_env  <- the environment texture, clamped

    # forward renderer only, from here down
    add_point -> programs "lmap_point" / "add_point", depth write off,
                 blend one:one, bind s_base and the point attenuation twice
    add_spot  -> programs "lmap_spot" / "add_spot", depth write off, blend one:one,
                 bind s_base, the spot cookie with projective division, the attenuation
    lighting_only -> programs "lmap_l", bind s_base, s_lmap, s_hemi
```

**Notes** — The dynamic-light and lighting-only elements are compiled out of the deferred builds, not merely unreached: those renderers have no such elements. The world-pass element is shared, which is why one file serves both generations here where most templates need two.

The hemispheric-ambient texture is sampled through the render-target sampler rather than a normal one, because it is a texture the renderer produced this frame, not an asset — it is clamped and unmipped.
