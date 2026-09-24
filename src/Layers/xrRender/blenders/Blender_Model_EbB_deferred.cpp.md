# src/Layers/xrRender/blenders/Blender_Model_EbB_deferred.cpp

> The same reflective-model template, filled for the deferred renderers: the reflection is dropped, the model becomes a g-buffer write, and only the blended case stays forward.

**Needs** — [`Blender_Model_EbB.h`](Blender_Model_EbB.h.md) · [`uber_deffer.h`](uber_deffer.h.md) · [`Blender.h`](../Blender.h.md) · [`Blender_Recorder.h`](../Blender_Recorder.h.md) · [`Blender_CLSID.h`](../Blender_CLSID.h.md) · [`Shader.h`](../Shader.h.md) · [Seam: Graphics device](../../../../SYSTEM-REQUIREMENTS.md#seam-graphics-device)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: it emits a pass description; the parameter block is a frozen tagged byte stream.

## Purpose

The deferred renderers answer the reflective-model class tag with this file instead of [`Blender_Model_EbB.cpp`](Blender_Model_EbB.cpp.md). Exactly one of the two is built. The parameter block, identity and comment are duplicated verbatim between them — the duplication is real and a rebuild should factor it out, keeping only `Compile` per generation.

The interesting decision is what happens to the reflection. In a deferred renderer there is no ambient pass to lay a reflection under, and the g-buffer has no channel for one, so **the environment map is simply not sampled**. The parameter is still loaded, because the shipped library contains it; it reaches nothing.

## `Compile`

**Contract** — two entirely different shapes, selected by the blend flag.

```text
FUNCTION compile(context)
  IF alpha_blend THEN
      # A translucent model cannot go through the g-buffer: the g-buffer
      # holds one surface per pixel. It is drawn forward, after the
      # deferred resolve, with the LOW-QUALITY forward program — the one
      # remaining place the environment map is still sampled.
      SELECT context.element
        normal_hq, normal_lq ->
          programs "model_env_lq"
          depth test and write on, blend src-alpha:inv-src-alpha, alpha test at 0
          bind s_base <- instance texture 0
          bind s_env  <- the env texture, clamped
      RETURN

  SELECT context.element
    normal_hq -> the shared deferred model emission, high quality
    normal_lq -> the shared deferred model emission, low quality
    shadow    -> the shadow-map pass: a depth-only draw with colour writes off
```

**Invariants**

- The deferred elements mark the stencil: value 1 written wherever the model covered a pixel, with a write mask that leaves the high bit alone. The deferred lighting passes test that stencil to skip pixels the g-buffer never covered, so **every template that writes the g-buffer must mark it the same way** or its pixels go unlit.
- The shadow element disables colour writes entirely. Hardware that can compare depth in the sampler needs no colour at all from a shadow pass; where the device exposes that, the pixel program is the do-nothing one.

**Notes** — Three variants of this file's `Compile` coexist, one per backend generation, differing only in how a texture reaches a sampler: the older deferred path binds a combined sampler-and-texture by one name, the newer ones bind the texture and the sampler state separately, and the portable path does the same with its own spelling. That difference belongs to the backend, not to the template, and a rebuild with a single binding model deletes all three copies in favour of one.

The blended path names the *low-quality* forward program even for the high-quality element. That is deliberate: the high-quality forward program samples the light projector, and the deferred renderers do not produce one.
