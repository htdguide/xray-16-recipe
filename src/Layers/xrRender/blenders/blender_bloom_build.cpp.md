# src/Layers/xrRender/blenders/blender_bloom_build.cpp

> The bloom chain's materials: one template whose five elements are the five steps of a separable blur into a half-resolution target, plus the final post-processing composite.

**Needs** — [`blender_bloom_build.h`](blender_bloom_build.h.md) · [`Blender.h`](../Blender.h.md) · [`Blender_Recorder.h`](../Blender_Recorder.h.md) · [`r2_types.h`](../../xrRender_R2/r2_types.h.md) · [Seam: Graphics device](../../../../SYSTEM-REQUIREMENTS.md#seam-graphics-device)
**Used by** — reached through its declarations in [`blender_bloom_build.h`](blender_bloom_build.h.md); callers name that, not this file.
**Tier floor** — T2: it emits a pass description with no parameter block and no data-driven identity.

## Purpose

Internal templates, constructed by the render target directly and named by nothing in the shipped data. Their class identifier is **zero** — the sentinel for "this template is not in the material library and will never be looked up by tag".

Their shape is the reason they are worth reading: this is where the recipe's blender machinery is used not as a *material* system but as a way of *naming a post-processing step*. The engine wants five different full-screen passes over five different render targets; rather than five templates, it makes one template with five elements, and the render target selects an element by index when it draws.

## The five elements

```text
0  build     read the generic scratch target, write the bloom target
             blend src-alpha : inv-src-alpha
1  X filter  read bloom target 1, write bloom target 2
2  Y filter  read bloom target 2, write bloom target 1
3  fast X    read bloom target 1, using the wide single-pass kernel
4  fast Y    read bloom target 2, using the wide single-pass kernel
```

**Invariants**

- Elements 1 and 2 are the **separable** blur: a horizontal pass and a vertical pass, ping-ponging between two targets. Elements 3 and 4 are the alternative — one wider kernel, named by a different pixel program, run twice. The render target chooses which pair to run; the templates offer both and decide nothing.
- The two bloom targets must be distinct and must alternate. The X filter reads target 1 and the Y filter reads target 2, so whichever the X pass wrote is what the Y pass reads; getting that wrong reads and writes one surface and produces a feedback smear.
- Every sampler in the chain is bound through the **render-target linear sampler**: clamped, linearly filtered, unmipped. Clamping is what keeps the blur from wrapping bright pixels from the opposite edge of the screen into frame.

## `CBlender_bloom_build`

**Contract** — the five elements above. No parameters, no save or load, no capabilities. Present in every renderer generation.

**Notes** — Each generation names a different vertex program for the same element: the oldest deferred path names the do-nothing program and relies on the caller having set up a pre-transformed quad, and the newer ones name an explicit pass-through program per element. That is a consequence of the newer devices having no fixed-function transform at all, not a difference in what the pass does.

## `CBlender_bloom_build_msaa`

**Contract** — the same five elements, existing only in the generations that support multisampling. Its emissions are identical to the non-multisampled template's in every generation that has it.

**Notes** — Nothing here distinguishes it from its non-multisampled twin. Either the distinction was intended and never implemented, or the bloom chain always reads an already-resolved target and never needed one; the source gives no way to tell. A rebuild should have one template until it finds a reason for two.

## `CBlender_postprocess_msaa`

**Contract** — the final composite: takes the scene, applies a noise texture and optionally a pair of colour-grading gradients, and writes the presented image. Two elements, not consecutive.

```text
element 0  programs "stub_notransform_postpr" / "postprocess"
           blend src-alpha : inv-src-alpha
           bind s_base0 and s_base1 <- the generic scene target (the SAME target twice)
           bind s_noise <- the shipped noise asset

element 4  programs "stub_notransform_postpr" / "postprocess_CM"
           the same three bindings, plus
           bind s_grad0 and s_grad1 <- the two colour-map targets
```

**Invariants**

- `s_base0` and `s_base1` are bound to the same target. The pixel program samples the scene twice — once sharp and once through an offset the vertex program supplies — and the two names exist so that a rebuild that wants to separate them can, without changing the shader source.
- The element indices are **0 and 4**, with 1 through 3 absent. The gap is not an accident: the render target selects the colour-graded variant by index, and the index it uses is the same one the bloom chain's "fast Y" step occupies in the other template. A rebuild that renumbers must renumber both call sites.
- The two colour-map gradients are named by the reserved user-target escapes, so the engine can swap them as the weather cycle advances without recompiling the material.

**Notes** — The noise texture is the one *asset* in this whole file: a shipped image, referenced by path, whose spelling is frozen. Everything else is a render target named by a reserved escape.
