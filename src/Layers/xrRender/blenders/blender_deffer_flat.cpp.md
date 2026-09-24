# src/Layers/xrRender/blenders/blender_deffer_flat.cpp

> The deferred renderers' filling for the lightmapped-diffuse class tag: the workhorse world surface, reduced to one g-buffer write and one shadow-map write.

**Needs** — [`blender_deffer_flat.h`](blender_deffer_flat.h.md) · [`uber_deffer.h`](uber_deffer.h.md) · [`Blender.h`](../Blender.h.md) · [`Blender_Recorder.h`](../Blender_Recorder.h.md) · [`Blender_CLSID.h`](../Blender_CLSID.h.md) · [`Shader.h`](../Shader.h.md) · [Seam: Graphics device](../../../../SYSTEM-REQUIREMENTS.md#seam-graphics-device)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: it emits a pass description; the parameter block is a frozen tagged byte stream.

## Purpose

The same class tag that [`BlenderDefault.cpp`](BlenderDefault.cpp.md) answers in the forward renderer. The shipped material library does not change; only the table mapping the tag to a class does, and that table is per backend. This is the single most important consequence of the material system's design, and this pair of files is where to see it: the same material name renders through a five-element multipass template in one renderer and a three-element deferred template in another.

What disappears in the deferred form is everything about *lighting*. There is no lightmap sampler, no hemispheric ambient, no additive point or spot pass. The surface writes its albedo, its normal and its position into the g-buffer, marks the stencil, and the lighting happens later for every surface at once.

## State

```text
RECORD Parameters
  tessellation : enum { none, pn_triangles, height_map, both }, default none
```

**Invariants** — detailable and **not** lightmappable, where the forward filling of the same tag answers yes to both. The lightmap answer differs between the two fillings of one tag, which is legitimate: the query is asked by the level compiler when baking and by the material compiler when compiling, and the deferred renderer genuinely does not sample a lightmap.

Also parallax-capable, which the forward filling is not.

## `Save` / `Load`

**Contract** — the tessellation selector with its four labels; version-gated, absent at version 0. The option count is re-asserted after reading.

## `Compile`

```text
FUNCTION compile(context)
  IF the device supports hardware tessellation THEN
      pass the tessellation selector through to the compile context
      # this is the ONE place the selector reaches anything

  SELECT context.element
    normal_hq ->
      shared deferred emission, high quality, programs "base" / "base"
      mark the stencil
    normal_lq ->
      shared deferred emission, low quality, programs "base" / "base"
      mark the stencil
    shadow ->
      the shadow-map write: a depth-only pass with colour writes disabled,
      binding the base texture for nothing but the alpha the shared shadow
      emission may need
```

**Invariants** — the stencil mark is 1 under a write mask that preserves the high bit, and the deferred resolve tests for it. Every g-buffer-writing template in this directory makes the same mark.

**Notes** — The tessellation selector is loaded by four different templates in this directory and reaches a shader in exactly one of them: this one, on exactly one backend. Everywhere else it is bytes in the parameter block and nothing more. A rebuild must still read those bytes.

Three copies of `Compile` exist, one per backend generation. They differ in three ways, all of them backend concerns rather than material ones: whether the stencil is marked at all (the oldest deferred renderer does not mark it and resolves the whole frame), whether the shadow pass names a do-nothing pixel program or a real one (hardware that compares depth in the sampler needs none), and how a texture and its sampler state reach the pass. One generation additionally routes the shadow pass through a shared shadow emission rather than naming the programs inline, because that generation supports tessellation in the shadow pass too and the displacement has to match the main pass exactly or the shadow detaches.
