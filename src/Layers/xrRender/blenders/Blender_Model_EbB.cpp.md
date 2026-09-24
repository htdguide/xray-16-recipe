# src/Layers/xrRender/blenders/Blender_Model_EbB.cpp

> The dynamic-model template with an environment reflection: the base texture's own alpha channel decides, per texel, how much of a reflection map shows through.

**Needs** — [`Blender_Model_EbB.h`](Blender_Model_EbB.h.md) · [`Blender.h`](../Blender.h.md) · [`Blender_Recorder.h`](../Blender_Recorder.h.md) · [`Blender_CLSID.h`](../Blender_CLSID.h.md) · [`Shader.h`](../Shader.h.md) · [`xrRender_console.h`](../xrRender_console.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: it emits a pass description; the parameter block is a frozen tagged byte stream.

## Purpose

"Env-by-base": the same model template as the default one, with a reflection layer underneath. The template's comment spells the composition — the environment map is laid down first, the base texture is laid over it, and **the base texture's alpha is the blend weight**. A texel with alpha 0 shows pure reflection, alpha 255 shows pure base. That convention is how the shipped art marks the shiny parts of a weapon or a helmet: not with a separate mask, but with the diffuse texture's own alpha.

The reflection texture is named by the template's own parameter, not by the material instance, and is sampled through a matrix whose mode makes it a reflection lookup rather than a uv transform (see the matrix modes in [`Blender_Recorder.cpp`](../Blender_Recorder.cpp.md)).

This is the forward renderer's filling; the deferred renderers use [`Blender_Model_EbB_deferred.cpp`](Blender_Model_EbB_deferred.cpp.md) for the same class tag.

## State

```text
RECORD Parameters
  env_texture : text, default "$null"   # the reflection map
  env_matrix  : text, default "$null"   # the matrix that generates its coordinates
  alpha_blend : bool, default false
```

**Invariants** — the parameter version is **forced to 1 on every save**, whatever it was on load. The blend flag was added at version 1 and the field must be present in anything this build writes; re-reading a version-0 record leaves the flag false.

## `Save` / `Load`

**Contract** — a marker then the environment texture, its matrix, and (from version 1) the blend flag. The marker is a named divider the authoring tools group parameters under; it occupies bytes and must be written.

## `Compile` — the fixed-function path

**Contract** — one pass, whose shape depends on the element *and* on whether the fallback lighting configuration is active. The fallback forces the low-quality shape and halves the final modulation.

```text
FUNCTION compile_fixed(context)
  fallback := lightmaps and dynamic lights are both off
  element  := fallback ? normal_lq : context.element
  modulate := fallback ? single : doubled

  pass:
    IF fallback AND alpha_blend THEN depth test on, depth write OFF, blend by alpha
                                ELSE depth test and write on, replace
    lighting and fog on

    SELECT element
      normal_hq ->
        stage 0: the light projector, through its own matrix
        stage 1: the environment map, clamped, through the env matrix
        stage 2: the base texture
        programs: no vertex program, pixel program "model_env"
        # three stages feeding one pixel program: the composition itself
        # lives in that program, not in the combine chain

      normal_lq ->
        stage 0: environment map, clamped, selected
        stage 1: base texture, BLENDED AGAINST THE RUNNING VALUE BY THE
                 TEXTURE'S OWN ALPHA; alpha passes the texture through
        stage 2: vertex colour, modulate (doubled unless in fallback)
```

**Invariants** — the environment map is always **clamped**. A reflection lookup addressed outside its range must saturate at the edge, not wrap; wrapping produces a seam that sweeps across the surface as the camera turns.

**Notes** — The low-quality shape is the one that states the convention explicitly, in the fixed-function combine op that blends by the texture's alpha. The high-quality shape hands the same three inputs to a pixel program and lets it decide — which is the same decision expressed where there is room for it.

## `Compile` — the programmable path

```text
FUNCTION compile_programmable(context)
  SELECT context.element
    normal_hq -> programs "model_env_hq"
                 IF alpha_blend THEN blend src-alpha:inv-src-alpha, alpha test at 0
                 bind s_base <- instance texture 0
                 bind s_env  <- the env texture, clamped
                 bind s_lmap <- the light projector, clamped, projective
    normal_lq -> programs "model_env_lq", same blend decision
                 bind s_base and s_env only
    add_point -> programs "model_def_point" / "add_point"   # shared with the plain model
                 depth write off, blend one:one, alpha test on
                 bind s_base, s_lmap and s_att <- point attenuation
    add_spot  -> programs "model_def_spot" / "add_spot"
                 same; s_lmap <- spot cookie with projective division, s_att <- attenuation
    lighting_only -> programs "model_def_shadow" / "model_shadow"
                     depth write off, blend zero : source colour, no textures
```

**Invariants** — the dynamic-light and shadow elements are *identical* to the plain model template's. The reflection is a property of the ambient pass only: a dynamic light adds light, and adding it to a reflection would reflect the light twice. Sharing the program names is how that is enforced — one shader source, two templates.

**Notes** — Where the plain model template passes its alpha reference into the light passes, this one hardcodes the reference to 0 in the base pass and takes the default in the light passes. The template has no alpha reference parameter at all, which is the real difference: a reflective surface is either fully blended or fully opaque, never cut out.
