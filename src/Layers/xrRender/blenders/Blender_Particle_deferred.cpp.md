# src/Layers/xrRender/blenders/Blender_Particle_deferred.cpp

> The particle material filled for the deferred renderers: the same six blend modes, plus soft-particle depth fading and a shadow-map element that turns every mode into a darkening pass.

**Needs** — [`Blender_Particle.h`](Blender_Particle.h.md) · [`Blender.h`](../Blender.h.md) · [`Blender_Recorder.h`](../Blender_Recorder.h.md) · [`Blender_CLSID.h`](../Blender_CLSID.h.md) · [`Shader.h`](../Shader.h.md) · [Seam: Graphics device](../../../../SYSTEM-REQUIREMENTS.md#seam-graphics-device)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: it emits a pass description; the parameter block is a frozen tagged byte stream.

## Purpose

The deferred renderers answer the particle class tag with this file instead of [`Blender_Particle.cpp`](Blender_Particle.cpp.md); exactly one of the two is built, and everything but `Compile` is duplicated verbatim.

Two things change. First, the replace mode gets its own deferred program and becomes a genuine g-buffer write — an opaque particle is lit like any other surface. Second, **every** mode binds the scene's position buffer as a second texture, which is what makes particles fade out as they approach the geometry behind them instead of cutting a hard line across it.

## `Compile`

```text
FUNCTION compile(context)
  SELECT context.element

    normal_hq, normal_lq ->
      SELECT blend_mode
        SET -> programs "deffer_particle", depth test on, depth WRITE ON,
               no blend, NO alpha test, reference 200 recorded anyway
        the other five -> exactly the forward renderer's state, program pair "particle"
      bind s_base     <- instance texture 0, clamped or wrapped per the flag
      bind s_position <- the scene position buffer, through the unfiltered sampler

    shadow ->
      # a particle in a shadow map cannot write depth and cannot add light:
      # all it can do is DARKEN the map, so every translucent mode collapses
      # to the same multiply and differs only in its pixel program.
      SELECT blend_mode
        SET       -> program pair "particle", depth write on, alpha test at 200
        BLEND     -> "particle-clip" / "particle_s-blend", blend dest-colour : zero
        ADD       -> "particle-clip" / "particle_s-add",   blend dest-colour : zero
        MUL       -> "particle-clip" / "particle_s-mul",   blend dest-colour : zero
        MUL_2X    -> "particle-clip" / "particle_s-mul",   blend dest-colour : zero
        ALPHA-ADD -> "particle-clip" / "particle_s-aadd",  blend dest-colour : zero
      bind s_base and s_position as above

    the fourth element -> emits nothing
```

**Invariants**

- The position buffer is sampled through the **unfiltered** sampler. It holds a depth or a view-space position per pixel, and interpolating between two such values across a depth discontinuity produces a position that is on neither surface — the soft fade would then halo around every silhouette.
- `MUL` and `MUL_2X` share one shadow program. In a shadow map there is no doubling to do: both modes mean "this particle removes light", and the factor is carried by the particle's own colour.
- The replace mode's deferred pass drops the alpha test that the forward one relies on, while still recording the reference 200. On the deferred path the cut-out is done in the pixel program instead, so the test would be redundant; the recorded reference is inert.

**Notes** — Three copies of `Compile` exist, one per backend generation, differing only in how a texture and its sampler state reach the pass. One of them additionally forces clamp addressing onto the resolved sampler after the fact, rather than requesting it at bind time, because on that backend the sampler object is shared and named separately from the texture — the same decision expressed where that backend allows it.

The fourth element, reserved in the source for an environment-map variant, emits nothing in every generation. Nothing requests it.
