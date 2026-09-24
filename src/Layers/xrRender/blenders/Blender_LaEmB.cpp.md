# src/Layers/xrRender/blenders/Blender_LaEmB.cpp

> A lightmapped surface with an additive environment layer, written twice over — once for hardware with two texture stages and once for three.

**Needs** — [`Blender_LaEmB.h`](Blender_LaEmB.h.md) · [`Blender.h`](../Blender.h.md) · [`Blender_Recorder.h`](../Blender_Recorder.h.md) · [`Blender_CLSID.h`](../Blender_CLSID.h.md) · [`Shader.h`](../Shader.h.md) · [`HWCaps.h`](../HWCaps.h.md) · [`xrRender_console.h`](../xrRender_console.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: it emits a pass description; the parameter block is a frozen tagged byte stream.

## Purpose

The composition the template's comment spells out is `(lightmap + environment * constant) * base`: a surface whose baked lighting is *added to* a reflection before both modulate the base texture. Used for wet, glossy or metallic world geometry where the reflection reads as a sheen over the lighting rather than as a mirror.

It is dead in the current engine — nothing constructs it and no shipped material names its tag in a level that still loads — but the tag is part of the frozen list.

The file's real content is not the composition, it is **the fitting**: the composition needs three texture stages, older hardware had two, and the optional constant multiplier needs another. Six variants exist, chosen by two independent facts.

Forward renderer only.

## State

```text
RECORD Parameters
  env_texture : text, default "$null"
  env_matrix  : text, default "$null"
  env_const   : text, default "$null"   # "$null" means "no constant multiplier"
```

**Invariants** — a constant name of exactly `"$null"` (compared case-insensitively) is the sentinel for *absent*, not the name of a constant. Whether it is absent changes which variant compiles, so the comparison is load-bearing.

## `Save` / `Load`

**Contract** — a marker, then the environment texture, its matrix, and its constant. No version gating; every version has all three.

## `Compile`

**Contract** — picks one of six emissions from two facts: whether a constant multiplier is named, and how many texture stages the device reports. Declares itself lightmappable, not detailable.

```text
FUNCTION compile(context)
  has_constant := env_const is not "$null"

  IF lightmaps and dynamic lights are both off THEN
      emit the authoring-tool shape (with or without the constant)
      RETURN

  SELECT context.element
    normal_hq, normal_lq ->
      IF the device reports exactly 2 texture stages THEN two-pass variant
                                                     ELSE three-stage variant
      (again, with or without the constant)
    lighting_only ->
      the lighting-only variant (with or without the constant)
    # no dynamic-light elements at all: this template takes no point or spot light
```

**Notes** — The stage count is read from the device's reported raster capabilities and only two values are distinguished: exactly two, or anything else. The named hardware in the source's comments dates the split precisely — two stages was the first generation of consumer transform-and-lighting parts, three was everything after. A rebuild targeting anything modern implements the three-stage variant and deletes the rest; the two-stage variants are recorded here only because they show *what the composition costs when it does not fit*.

## The variants

**Contract** — all six compute the same result; they differ in how many passes it takes.

```text
# three stages, no constant — the reference shape, one pass
  stage 0: the lightmap
  stage 1: the environment map, ADDED to it
  stage 2: the base texture, DOUBLED against the sum

# three stages, with constant — same, reordered so the constant has a stage
  stage 0: the environment map * the pass constant
  stage 1: the lightmap, ADDED
  stage 2: the base texture, doubled against the sum

# two stages, no constant — two passes
  pass 0: lightmap, then environment added          (depth write on, replace)
  pass 1: base texture, DOUBLED INTO THE FRAME      (depth write off, multiply-2x blend)

# two stages, with constant — two passes, the constant taking stage 0
  pass 0: environment * constant, then lightmap added
  pass 1: base texture, doubled into the frame

# lighting-only, either form — one pass, the lighting terms alone,
#   no base texture, no fog
```

**Invariants**

- Where the composition does not fit in one pass, the *multiply* is moved out of the combine chain and into the **frame-buffer blend**: the second pass multiplies what pass 0 left in the frame by the base texture, doubled. That is the general trick, and it is why the second pass turns depth writing off and the first leaves it on.
- In the constant-carrying variants the lightmap is sampled through the positional name for the material's second texture with the second uv channel, rather than through the canonical lightmap stage. The canonical stage also clamps the addressing; these do not. That difference is not explained anywhere and may be an oversight — lightmap charts are authored with a border, so wrapping rarely shows.

**Notes** — The authoring-tool shape with a constant is the only place the composition is written with the *diffuse* vertex colour standing in for the lightmap. In the tools there is no baked lightmap yet, so the interactive preview substitutes vertex lighting; the shape is otherwise the two-pass one.
