# src/Layers/xrRender/blenders/Blender_BmmD_deferred.cpp

> The terrain template filled for the deferred renderers: a four-way ground blend driven by a mask texture derived from the base texture's name.

**Needs** — [`Blender_BmmD.h`](Blender_BmmD.h.md) · [`uber_deffer.h`](uber_deffer.h.md) · [`Blender.h`](../Blender.h.md) · [`Blender_Recorder.h`](../Blender_Recorder.h.md) · [`Blender_CLSID.h`](../Blender_CLSID.h.md) · [`Shader.h`](../Shader.h.md) · [Seam: Graphics device](../../../../SYSTEM-REQUIREMENTS.md#seam-graphics-device)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: it emits a pass description; the parameter block is a frozen tagged byte stream.

## Purpose

The deferred renderers answer the terrain class tag with this file instead of [`Blender_BmmD.cpp`](Blender_BmmD.cpp.md). Exactly one of the two is built; everything but `Compile` is duplicated between them verbatim.

What this filling adds is the feature the four channel parameters exist for. The high-quality element blends **four different ground textures** across the terrain, weighted per texel by the four channels of a mask, with a normal map for each. The result is grass fading into asphalt fading into earth without the terrain needing a unique texture per region.

## The derived-name convention

**This is the part a rebuild must reproduce exactly.** None of these textures is named in the data; every one is spelled from a name that is.

```text
mask_texture      := <the material's base texture name> + "_mask"
normal_for(t)     := t + "_bump"
parallax_for(t)   := t + "_bump" + "#"
```

So a terrain material whose base texture is `terrain_swamp` implies a mask at `terrain_swamp_mask`, and a ground channel named `detail/detail_grnd_grass` implies a normal map at `detail/detail_grnd_grass_bump` and, when steep parallax is enabled, a height map at `detail/detail_grnd_grass_bump#`. The suffixes `_mask`, `_bump` and the trailing `#` are **frozen**: the files ship under those names and nothing declares them.

## `Compile`

```text
FUNCTION compile(context)
  mask := base texture name + "_mask"

  SELECT context.element
    normal_hq ->
      shared deferred emission, high quality, base programs "impl"/"impl",
        with the template's detail texture substituted for the resolved one
      bind s_mask <- mask
      bind s_lmap <- instance texture 1
      bind s_dt_r, s_dt_g, s_dt_b, s_dt_a <- the four channel textures,
           WRAPPED and anisotropically filtered
      bind s_dn_r, s_dn_g, s_dn_b, s_dn_a <- their derived normal maps
      IF steep parallax is in effect THEN
          bind s_dn_rX, s_dn_gX, s_dn_bX, s_dn_aX <- their derived height maps
      mark the stencil

    normal_lq ->
      shared deferred emission, low quality, base programs "base"/"impl"
      bind s_lmap <- instance texture 1
      mark the stencil
      # no mask, no channels: the four-way blend is the high-quality feature

    shadow ->
      a depth-only pass with colour writes off
```

**Invariants**

- The four ground textures are the only samplers in the engine bound with **wrap addressing and anisotropic filtering explicitly stated**. Wrap because they tile densely across a terrain sector; anisotropic because ground is almost always viewed at a grazing angle, and it is the one place where the default filtering visibly smears.
- The stencil mark is value 1 under a write mask that preserves the high bit — the same mark every g-buffer-writing template makes, and what the deferred lighting passes test against.

**Notes** — Three copies of `Compile` exist, one per backend generation, differing only in how a texture reaches a sampler and in whether the stencil mark is emitted at all (the oldest deferred path omits it). One of the three has the whole four-channel binding block commented out and replaced with the separate-texture-and-sampler spelling — the same bindings, written the other way. A rebuild with one binding model keeps one copy.
