# src/Layers/xrRender_R2/r2_blenders.cpp

> Maps a material's class identifier — a tag frozen in the shipped level and model data —
> to the deferred pass description that implements it.

**Needs** — [`r2.h`](r2.h.md) ·
[`xrRender/Blender_CLSID.h`](../xrRender/Blender_CLSID.h.md) ·
[`xrRender/blenders/blender_deffer_flat.h`](../xrRender/blenders/blender_deffer_flat.h.md) ·
[`xrRender/blenders/blender_deffer_aref.h`](../xrRender/blenders/blender_deffer_aref.h.md) ·
[`xrRender/blenders/blender_deffer_model.h`](../xrRender/blenders/blender_deffer_model.h.md) ·
[`xrRender/blenders/Blender_BmmD.h`](../xrRender/blenders/Blender_BmmD.h.md) ·
[`xrRender/blenders/Blender_tree.h`](../xrRender/blenders/Blender_tree.h.md) ·
[`xrRender/blenders/Blender_detail_still.h`](../xrRender/blenders/Blender_detail_still.h.md) ·
[`xrRender/blenders/Blender_Particle.h`](../xrRender/blenders/Blender_Particle.h.md) ·
[`xrRender/blenders/Blender_Model_EbB.h`](../xrRender/blenders/Blender_Model_EbB.h.md) ·
[`xrRender/blenders/Blender_Screen_SET.h`](../xrRender/blenders/Blender_Screen_SET.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: a lookup table.

## Purpose

A material file in the game data declares its *class* by a four-character tag. The tag
selects an object that knows how to emit the material's passes for the renderer in use.
The same tag therefore means something different on each renderer, and this file is the
deferred path's answer. It is the seam between frozen data and this chapter's pass
vocabulary.

## State

Stateless.

## `create_blender`

**Contract** — given a class identifier, return a newly created pass-description object,
or nothing when this renderer has no implementation for that class. The caller owns the
result and destroys it through the companion entry point. Never fails.

```text
FUNCTION create_blender(class_id) -> optional<PassDescription>
  # --- surfaces that become G-buffer writers ---
  default, vertex-lit            -> deferred flat
  default with alpha test        -> deferred with alpha test, "tested" variant
  vertex-lit with alpha test     -> deferred with alpha test, "untested" variant
  model                          -> deferred model
  model with environment bump    -> model, environment-mapped
  bump with detail (two spellings, lightmapped and not) -> bump-with-detail
  lightmapped environment bump   -> lightmapped environment bump
  tree                           -> tree (wind animation in the vertex stage)
  detail                         -> detail (the grass layer)
  particle                       -> particle

  # --- screen-space and editor ---
  screen set                     -> screen-set (blit a named target)
  editor wire / selection        -> their editor materials

  # --- classes this renderer does not implement ---
  screen gray, light, shadow texture, shadow world, blur,
  ambient-plus-environment-bump, plain bump      -> nothing

  otherwise                      -> nothing
```

**Notes** — the classes that return nothing are the fixed-function renderer's vocabulary:
explicit light passes, explicit shadow passes, a blur. On a deferred path lighting and
shadowing are not properties of a material at all, so there is nothing to emit; the
material system treats "no description" as "this surface does not draw on this renderer",
which is exactly right for a shadow-volume material in a shadow-map world.

Two class identifiers map to the *same* deferred description (default and vertex-lit both
become deferred flat) because the distinction they encode — whether lighting comes from a
lightmap or from per-vertex colour — is meaningless once all lighting is deferred. The
alpha-tested pair keeps its distinction only to choose whether the alpha reference is
taken from the material or from the engine.
