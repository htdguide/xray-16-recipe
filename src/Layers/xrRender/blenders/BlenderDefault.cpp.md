# src/Layers/xrRender/blenders/BlenderDefault.cpp

> The workhorse world-surface template for the forward renderer: a base texture modulated by a baked lightmap, plus the additive passes that add a dynamic point or spot light on top of it.

**Needs** — [`BlenderDefault.h`](BlenderDefault.h.md) · [`Blender.h`](../Blender.h.md) · [`Blender_Recorder.h`](../Blender_Recorder.h.md) · [`Blender_CLSID.h`](../Blender_CLSID.h.md) · [`Shader.h`](../Shader.h.md) · [`xrRender_console.h`](../xrRender_console.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: it emits a pass description. What stops T3 is the parameter block, which is read from the shipped material library as a tagged byte stream.

## Purpose

Most of the static geometry in a level is drawn by this template. The name in the shipped data is the lightmapped-diffuse identifier, and what it means is: the surface's colour is its base texture, its lighting is a baked lightmap sampled from a second texture, its ambient sky contribution comes from a third, and a detail texture may be layered on top if the texture-description database pairs one with the base.

This file is the **forward renderer's** filling. The deferred renderers answer the same identifier with a different template ([`blender_deffer_flat.cpp`](blender_deffer_flat.cpp.md)); the shipped material library does not change, only the table that maps the identifier to a class. The file refuses to compile into any other renderer generation, which is the build system's way of stating that.

## State

```text
RECORD Parameters
  tessellation : enum { none, pn_triangles, height_map, both }   # default none
```

**Invariants** — the template declares itself **detailable** and **lightmappable**: the first lets the material compiler resolve a detail texture from the base texture's description, the second tells the level compiler this surface takes a baked lightmap. Both answers are part of the data contract, not internal policy.

## `Save` / `Load`

**Contract** — writes and reads the parameter block. Version 0 has no block at all; version 1 adds the tessellation selector, written as an enumerated token with its four labels inlined after it. A version-0 record must be accepted and left at the default.

**Notes** — The four labels are written into the file next to the value, so the authoring tools can present a menu without knowing the enumeration. They are dead weight to the game, which only reads the selected index — but they occupy bytes in the shipped library and a rebuild that writes one must write them.

## `Compile` — the fixed-function path

**Contract** — emits the pass list for hardware without programmable shading. Two shapes, chosen by whether lightmaps and dynamic lights are enabled at all.

```text
FUNCTION compile_fixed(context)
  IF lightmaps and dynamic lights are both off THEN
      # the authoring-tool / fallback shape: one pass, one stage,
      # the base texture modulated by the interpolated vertex colour,
      # with fixed-function lighting and fog both on
      pass:
        stage 0: texture * vertex colour, both channels
      RETURN

  SELECT context.element
    normal_hq, normal_lq ->
      pass:
        depth test and write on, no blending, fog on and lighting off
        stage 0: the lightmap, if lightmaps are on   # canonical lightmap stage
        stage 1: base texture, DOUBLED against the running value
    lighting_only ->
      pass:
        depth test and write on, no blending, no lighting and no fog
        stage 0: the lightmap only
```

**Notes** — The doubling in the second stage is the engine's lightmap convention: lightmaps are stored at half intensity so that a surface can be lit *brighter* than its texture, and every template that samples one multiplies by two on the way out. A rebuild that multiplies by one will render every shipped level at half brightness.

The lightmap stage is emitted conditionally even in the lightmapped path, because the oldest hardware could be told to skip lightmaps entirely as a performance setting; the pass then degenerates to the base texture alone rather than to nothing.

## `Compile` — the programmable path

**Contract** — emits one pass per shader element, naming a pair of shader programs and binding samplers by name. Requires the material instance to name **at least three textures**: base, lightmap, and the hemispheric-ambient lookup. Fewer is fatal.

```text
FUNCTION compile_programmable(context)
  SELECT context.element

    normal_hq ->                        # the full-quality world pass
      program pair := detail_diffuse ? "lmap_dt" : "lmap"
      bind s_base  <- instance texture 0
      bind s_lmap  <- instance texture 1
      bind s_hemi  <- instance texture 2     through the render-target sampler
      IF detail_diffuse THEN bind s_detail <- the resolved detail texture
      fog on, fixed-function lighting off

    normal_lq ->                        # same, detail layer never applied
      program pair := "lmap"
      bind s_base, s_lmap, s_hemi as above

    add_point ->                        # one dynamic point light, additive
      programs "lmap_point" / "add_point"
      depth test on, depth WRITE OFF, blend one:one, alpha test on at reference 0
      bind s_base <- instance texture 0
      bind s_lmap and s_att <- the shared point-light attenuation texture

    add_spot ->                         # one dynamic spot light, additive
      programs "lmap_spot" / "add_spot"
      depth test on, depth write off, blend one:one, alpha test on at reference 0
      bind s_base <- instance texture 0
      bind s_lmap <- the shared spot cookie, WITH PROJECTIVE DIVISION ENABLED
      bind s_att  <- the shared spot attenuation texture

    lighting_only ->                    # the lightmap-capture element
      programs "lmap_l"
      bind s_base, s_lmap, s_hemi
```

**Invariants**

- The additive light passes write no depth. They are drawn after the surface has already laid its depth down, and a second write from a blending pass fights itself.
- The spot pass is the only one that asks for projective division on a sampler: the spot cookie is addressed by a projected coordinate, and the division must happen in the sampler because the coordinate arrives from the vertex program already in clip space.
- The sampler names — `s_base`, `s_lmap`, `s_hemi`, `s_att`, `s_detail`, `smp_rtlinear` — are matched against the *shipped shader source's* identifiers. They are frozen. A binding whose name the compiled programs do not use is silently dropped, which is how one template serves several shader generations.

**Notes** — The high- and low-quality elements differ only in whether the detail layer is present, and the program names differ only by a suffix. That is the pattern across the whole blender set: quality is a *different shader source file*, not a branch inside one, because the hardware generation this targeted could not afford the branch.
