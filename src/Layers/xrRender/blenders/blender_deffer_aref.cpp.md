# src/Layers/xrRender/blenders/blender_deffer_aref.cpp

> The deferred renderers' filling for the alpha-tested world surface, with two lives: a g-buffer write when it is cut out, and a forward-blended pass when it is translucent.

**Needs** — [`blender_deffer_aref.h`](blender_deffer_aref.h.md) · [`uber_deffer.h`](uber_deffer.h.md) · [`Blender.h`](../Blender.h.md) · [`Blender_Recorder.h`](../Blender_Recorder.h.md) · [`Blender_CLSID.h`](../Blender_CLSID.h.md) · [`Shader.h`](../Shader.h.md) · [`r2_types.h`](../../xrRender_R2/r2_types.h.md) · [Seam: Graphics device](../../../../SYSTEM-REQUIREMENTS.md#seam-graphics-device)
**Used by** — reached through its declarations in [`blender_deffer_aref.h`](blender_deffer_aref.h.md); callers name that, not this file.
**Tier floor** — T2: it emits a pass description; the parameter block is a frozen tagged byte stream.

## Purpose

The deferred filling of the same class tag [`Blender_default_aref.cpp`](Blender_default_aref.cpp.md) answers in the forward renderer. It is the one template here constructed **two ways from one class**: a flag on the constructor says whether this instance is lightmapped, and the renderer registers the class twice, once under each setting, against two different tags.

That is how one implementation covers both the lightmapped and the vertex-lit alpha-tested surfaces. The flag changes exactly one thing — what the blended path samples — and is otherwise inert.

## State

```text
RECORD Parameters
  alpha_ref   : int in [0,255], default 200    # NOT 32: see Invariants
  alpha_blend : bool, default false

RECORD ConstructionFlag
  lightmapped : bool     # set at registration, not loaded from data
```

**Invariants** — the default alpha reference is **200**, where the forward filling of the same tag defaults to 32. A material saved without a parameter block therefore cuts out hard in the deferred renderer and softly in the forward one. The deferred default is the one that matches the foliage and detail-object references, which is what most alpha-tested world geometry is; the discrepancy is not reconciled anywhere and a rebuild has to keep both, because both are what their respective renderers do with the shipped data.

The capability answers depend on the construction flag: detailable and parallax-capable always, lightmappable only when the flag is set.

## `Save` / `Load`

**Contract** — the alpha reference then the blend flag. Reading is gated on version **exactly 1** — any other version reads nothing at all and leaves both defaults, which differs from the usual "at least" gating and means a future version would silently lose its parameters.

## `Compile` — the blended path

**Contract** — a translucent surface cannot be written into a g-buffer, which holds one surface per pixel. It is drawn forward instead, after the resolve, and the lightmapped flag selects which forward program.

```text
FUNCTION compile_blended(context)
  SELECT context.element
    normal_hq, normal_lq ->
      IF lightmapped THEN
          programs "lmapE" / "lmapE"
          blend src-alpha : inv-src-alpha, alpha test at alpha_ref
          bind s_base <- instance texture 0
          bind s_lmap <- instance texture 1
          bind s_hemi <- instance texture 2, through the render-target sampler
          bind s_env  <- the first environment map of the weather pair, clamped
      ELSE
          programs "vert" / "vert"
          same blend and alpha test
          bind s_base only
    shadow -> emit nothing
```

**Invariants** — a blended surface casts no shadow: the shadow element is absent from this path entirely. That is the same decision the forward filling makes when it refuses to add dynamic light to a blended surface, for the same reason — the surface is not opaque, so treating it as an occluder is wrong.

The lightmapped variant reaches for `s_env` from the *global* environment pair rather than from a material parameter. It is borrowing the reflective template's shader, which expects an environment map, and the weather cycle's sky reflection is the sensible thing to give it.

## `Compile` — the cut-out path

```text
FUNCTION compile_cutout(context)
  force the sort parameters: priority 1, no strict back-to-front
  atoc := the device resolves alpha tests through coverage

  SELECT context.element
    normal_hq, normal_lq ->
      IF atoc THEN
          shared deferred emission, pixel program "base_atoc"
          mark the stencil, disable colour writes, enable alpha-to-coverage
          end the pass
      shared deferred emission, pixel program "base"
      mark the stencil
      IF atoc THEN require depth EQUAL
      end the pass
    shadow ->
      programs "shadow_direct_base_aref" / "shadow_direct_base_aref"
      colour writes disabled, base texture bound for the cut-out
```

**Invariants**

- The sort parameters are **overwritten** here, after the base template already wrote whatever the material asked for: priority 1, strict ordering off. A cut-out surface is opaque and must sort with the opaque geometry whatever the material claims; letting it sort back-to-front would cost a full sort for no benefit and would break the depth-equality trick the coverage path relies on. The source flags the line as suspect and does it anyway.
- The coverage path's second pass tests depth for equality against the first, confining the g-buffer write to the samples the coverage pass accepted — the same two-pass shape the foliage and detail-object templates use.

**Notes** — Three copies of `Compile` exist, one per backend generation, differing in whether the coverage path exists at all and in how textures reach samplers. The oldest deferred renderer has neither the coverage path nor the stencil mark, and chooses its shadow pass's pixel program on whether the device can compare depth in the sampler; the newer ones always use the cut-out shadow program and simply turn colour writes off.
