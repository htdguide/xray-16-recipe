# src/Layers/xrRender/blenders/uber_deffer.cpp

> Derives a g-buffer pass — which shader variant, which textures, which sampler settings — from the *names* of the textures a material happens to reference.

**Needs** — [`uber_deffer.h`](uber_deffer.h.md) · [`Blender_Recorder.h`](../Blender_Recorder.h.md) · [`ResourceManager.h`](../ResourceManager.h.md) · [`TextureDescrManager.h`](../TextureDescrManager.h.md) · [`r2_types.h`](../../xrRender_R2/r2_types.h.md) · [Seam: Graphics device](../../../../SYSTEM-REQUIREMENTS.md#seam-graphics-device)
**Used by** — [`uber_deffer.h`](uber_deffer.h.md)
**Tier floor** — T1: it composes shader-source filenames by string concatenation and probes the virtual filesystem for their existence, and it depends on the engine's texture-naming conventions being exactly as shipped.

## Purpose

Every surface that writes into the g-buffer — world geometry, models, terrain — needs the same pass built in the same way, and which of roughly forty shader variants it needs depends on four independent facts about the material: does its base texture have a companion bump map, does it carry a lightmap, does a detail texture apply, is it alpha-tested. Rather than write a blender per combination, one routine assembles the variant *name* from those facts and emits the single pass.

This is the reason the shipped data needs no per-material shader assignment: the **naming convention is the assignment**. A rebuild must reproduce the convention exactly, because it is the shader filenames on disk that it resolves to.

## `uber_deffer(context, hq, vertex_spec, pixel_spec, alpha_test, detail_override, do_not_finish)`

**Contract** — appends one g-buffer pass to the context: a vertex/pixel program pair chosen by name, five to nine texture bindings, and the sampler settings for them. Reads the material's texture list, the engine's texture-description table (which records which base textures have bump companions), and the renderer's detail-texture decisions already made by the recorder. Probes the virtual filesystem once, and only in one narrow case. Closes the pass unless `do_not_finish` is set, which lets a caller append stencil or coverage state before closing.

**Invariants** — the caller has already filled the context's detail-texture fields; `uber_deffer` consumes them and does not re-derive them.

### The four facts, and how each is decided

```text
FUNCTION derive_facts(C, detail_override) -> facts
  # 1. bump — does the base texture have a companion normal map?
  base = normalize_texture_name(C.textures[0])   # strips the engine's name decorations
  bump = texture_has_bump_companion(base)
  IF bump THEN
    bump_name  = bump_companion_of(base)
    bump_nameX = bump_name + "#"      # the SECOND half of the bump pair

  # 2. lmap — is this surface lightmapped?
  lmap = (C.textures has at least 3 entries)
         AND (C.textures[2] starts with the four characters "lmap")

  # 3. detail — which texture, if any, is layered on top
  detail = detail_override OR C.detail_texture OR ""

  # 4. detail bump — does THAT texture have a bump companion?
  detail_bump = C.wants_detail_bump AND texture_has_bump_companion(detail)
  IF detail_bump THEN
    detail_bump_name  = bump_companion_of(detail)
    detail_bump_nameX = detail_bump_name + "#"
  RETURN facts
```

**Invariants**

- The bump-map pair convention is frozen and shared with the shipped art: a normal map is stored as **two** textures, the named one and the same name with `"#"` appended. The first carries the normal's two components plus the height; the second carries the extra channels the engine's bump model needs. Both must be bound, and the sampler named `s_bumpX` **must be bound before** `s_bump`. The ordering is not cosmetic: on the older binding model these resolve to consecutive stages and the recorder allocates them in call order.
- Lightmap detection is a *prefix match on the third texture's name*, four characters, case-sensitively as the data spells it. There is no flag anywhere saying "this material is lightmapped"; the level compiler names the texture `lmap…` and that is the entire signal. The third slot is also where the hemisphere term arrives for non-lightmapped surfaces, which is why the test is a name test rather than a presence test.

### Assembling the variant name

```text
FUNCTION shader_names(facts, hq, vertex_spec, pixel_spec, alpha_test) -> (vs, ps)
  vs = "deffer_" + vertex_spec + (lmap ? "_lmh" : "")
  ps = "deffer_" + pixel_spec  + (lmap ? "_lmh" : "")
  IF alpha_test THEN ps = ps + "_aref"

  IF NOT bump THEN
    vs = vs + "_flat" ; ps = ps + "_flat"
    IF hq AND (detail_diffuse OR detail_bump_wanted) THEN
      vs = vs + "_d" ; ps = ps + "_d"
    IF hq AND steep_parallax AND pixel_spec == "impl" THEN
      IF a shader source file named ps + "_steep" exists THEN ps = ps + "_steep"
  ELSE
    vs = vs + "_bump"
    ps = ps + (hq AND steep_parallax ? "_steep" : "_bump")
    IF hq AND (detail_diffuse OR detail_bump_wanted) THEN
      vs = vs + "_d"
      ps = ps + (detail_bump ? "_db" : "_d")
    vs = vs + "-hq" ; ps = ps + "-hq"        # only ever appended when bump AND hq
  RETURN (vs, ps)
```

**Invariants**

- The suffixes are appended in exactly this order and the resulting string must name a file in the shipped shader set. The order is therefore frozen: `_lmh`, `_aref`, `_flat`/`_bump`/`_steep`, `_d`/`_db`, `-hq`.
- `-hq` is appended **only** in the bump branch. A flat high-quality surface and a flat low-quality surface differ only by the presence of `_d`, because without a normal map there is nothing extra for a high-quality variant to do.
- The steep-parallax probe is the one place this routine touches the filesystem, and it is a *test for existence*, not a load: parallax variants ship for only some surface kinds, and a missing file must degrade to the non-parallax variant rather than fail. It is gated on the pixel specification being literally `"impl"` — the implicit-geometry (terrain-like) family — because those are the only flat surfaces that ship a steep variant.

### The bindings

```text
FUNCTION bind(C, facts)
  C.texture("s_base",   C.textures[0])
  C.texture("s_bumpX",  bump_nameX)        # MUST precede s_bump
  C.texture("s_bump",   bump_name)
  C.texture("s_bumpD",  detail)            # the detail texture, read as a bump source
  C.texture("s_detail", detail)            # ...and again as a colour source
  IF detail_bump THEN
    C.texture("s_detailBump",  detail_bump_name)
    C.texture("s_detailBumpX", detail_bump_nameX)
  IF lmap THEN
    C.texture("s_hemi", C.textures[2])     # clamped, linear, no mip
```

All of the surface textures are sampled wrapped, anisotropically, with linear mip filtering; the lightmap/hemisphere texture is clamped and unmipped because it is a per-surface atlas page and bleeding across its edge is visible.

**Notes** — Binding the same detail texture under two names is not redundancy: the two samplers are read with different swizzles in the shader (colour modulation versus a derived bump), and on the older binding model each name resolves to its own stage with its own filtering. Merging them in a rebuild is safe only if the shader side is rewritten to match.

When a bump-mapped, high-quality surface is drawn on a device with hardware tessellation *and* the material asked for a tessellation method, the pass becomes a tessellated one: the variant name is rebuilt from scratch (it always contains `bump`, and the feature-suffix set is expressed as compile-time defines rather than filename suffixes), and hull and domain programs join the pair. The defines set is `TESS_PN`, `TESS_HM`, `USE_LM_HEMI`, `USE_TDETAIL`, `USE_TDETAIL_BUMP`, and the same list is also appended to the shader name in parentheses so that the compiled-blob cache keys on it. That parenthesised suffix is part of the cache key, not part of the filename on disk.

## `uber_shadow(context, vertex_spec)`

**Contract** — the same derivation, reduced to what a shadow-map pass needs: no colour, no lightmap sampling, no detail colour. Emits a depth-only pass. Exists only on the tessellating backend, because on every other one the shadow pass is a single fixed name and the blenders emit it directly.

**Notes** — It re-derives the bump and lightmap facts even though a depth-only pass does not shade, because a tessellated shadow pass displaces geometry from the height channel and therefore needs the same bump textures and the same defines as the colour pass. Without tessellation it collapses to one pass with a fixed name and the derivation is dead work — which is why it is compiled out elsewhere.
