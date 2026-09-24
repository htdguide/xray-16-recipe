# src/Layers/xrRender/blenders/blender_combine.cpp

> The deferred resolve: the material that reads every g-buffer channel plus the accumulated lighting and turns them into a lit image, and the four variants that composite bloom and distortion on top of it.

**Needs** — [`blender_combine.h`](blender_combine.h.md) · [`Blender.h`](../Blender.h.md) · [`Blender_Recorder.h`](../Blender_Recorder.h.md) · [`r2_types.h`](../../xrRender_R2/r2_types.h.md) · [Seam: Graphics device](../../../../SYSTEM-REQUIREMENTS.md#seam-graphics-device)
**Used by** — reached through its declarations in [`blender_combine.h`](blender_combine.h.md); callers name that, not this file.
**Tier floor** — T2: it emits a pass description with no parameter block and no data-driven identity.

## Purpose

This is the centre of the deferred renderer expressed as a material. Everything the g-buffer passes wrote and everything the light passes accumulated meets here, in one full-screen pass, and comes out as a lit image. Internal: class tag zero, constructed by the render target, named by nothing in the shipped data.

Its six elements are two different jobs. Element 0 is the resolve. Elements 1 through 4 are the *second* full-screen pass — antialiasing, bloom and screen distortion composited together — in the four combinations of "detect and smooth edges or not" by "apply distortion or not". Element 5 is reserved and empty.

## Element 0 — the resolve

**Contract** — reads eleven inputs and writes the lit scene. Blended inverse-source-alpha over source-alpha, which is the sky's compositing rule: the g-buffer's alpha carries "how much of this pixel is sky", and the resolve blends the lit surface against whatever the sky pass already put in the frame.

```text
FUNCTION element_0(context)
  programs "combine_1" / the generation's combine pixel program
  blend inverse-source-alpha : source-alpha
  stencil: pass only where the stencil is at least 1

  bind s_position    <- the g-buffer position target
  bind s_normal      <- the g-buffer normal target
  bind s_diffuse     <- the g-buffer albedo target
  bind s_accumulator <- the light accumulation target
  bind s_depth       <- the depth target
  bind s_tonemap     <- this frame's resolved average luminance
  bind s_material    <- the material lookup volume, through the material sampler
  bind s_occ         <- the ambient-occlusion scratch target
  bind s_half_depth  <- the half-resolution depth used by the occlusion pass
  bind env_s0, env_s1 <- the two environment maps the weather cycle blends between
  bind sky_s0, sky_s1 <- the two sky maps likewise
  bind the dither texture set
```

**Invariants**

- The **stencil test is the cull**. Every template that writes the g-buffer marks the stencil with 1; this pass runs only where that mark exists, so pixels the g-buffer never covered — sky, and anything outside the scene — are skipped entirely rather than resolved to black. A rebuild that drops the stencil mark from any g-buffer template will see that geometry vanish here.
- The environment and sky pairs are bound **two at a time** because the weather cycle interpolates between two keyframes; the blend weight arrives as a constant and the resolve does the interpolation per pixel rather than the engine doing it per frame on the CPU.
- The material lookup is a three-dimensional table indexed by the light-to-normal angle, the light-to-half-vector angle, and the surface material coordinate the texture-description database supplied. It is sampled through a dedicated sampler whose third axis wraps and whose first two clamp — so a material index past the last entry wraps to the first rather than clamping onto it.
- The dither texture set exists to break up banding in the accumulated lighting, which is stored at reduced precision.

## Elements 1–4 — the composite

**Contract** — four variants of one full-screen pass, differing only in the pixel program named.

```text
element 1  "combine_2_AA"     edge-detecting antialias, no distortion
element 2  "combine_2_NAA"    no antialias, no distortion
element 3  "combine_2_AA_D"   edge-detecting antialias, with distortion
element 4  "combine_2_NAA_D"  no antialias, with distortion

all four bind:
  s_position, s_normal    <- the g-buffer, for the edge detector
  s_image                 <- the lit scene from element 0
  s_bloom                 <- the finished bloom target
  s_distort               <- the scratch target the distortion pass wrote
```

**Invariants** — the antialiasing is an **edge detector over the g-buffer**, not a multisample resolve: it compares neighbouring positions and normals and blurs where they disagree. That is why the two g-buffer channels are still bound in a pass that has otherwise finished with them, and it is why the non-antialiased variants still bind them — the bindings are identical across all four and only the program differs.

All four bind the distortion target, including the two variants whose programs do not sample it. A binding the compiled programs do not name is dropped, so the uniformity costs nothing and keeps the four cases one code path.

## `CBlender_combine_msaa`

**Contract** — the same six elements against a multisampled frame. Carries two strings — a name and a definition — and, if the name is set, **parses the definition as an integer and writes it into the renderer's current-sample index** before compiling, restoring it to "all samples" afterwards.

**Notes** — That is the one genuinely unusual thing in this file: a template that mutates renderer state during compilation. It exists because per-sample resolve needs one compiled pass *per sample index*, and the sample index has to reach the shader compiler as a macro definition rather than as a constant. The template is instantiated once per sample with the index as a string, and the global it sets is what the shader compiler reads when it builds the macro set. A rebuild with a compiler that takes an explicit macro set passes the index directly and deletes the global.

The multisampled variants read the *resolved* distortion target rather than the raw one, which is the only binding that differs from the non-multisampled template.
