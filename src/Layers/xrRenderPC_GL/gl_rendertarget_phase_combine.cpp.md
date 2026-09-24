# src/Layers/xrRenderPC_GL/gl_rendertarget_phase_combine.cpp

> The OpenGL filling of step 11: the tone-mapped resolve, the forward pass, volumetric
> compositing, bloom, distortion, depth of field and motion blur — the same order as the
> other filling, over a different set of allocations.

**Needs** — [`gl_rendertarget.h`](gl_rendertarget.h.md) ·
[`gl_rendertarget_u_set_rt.cpp`](gl_rendertarget_u_set_rt.cpp.md) ·
[`gl_rendertarget_phase_flip.cpp`](gl_rendertarget_phase_flip.cpp.md) ·
[`../xrRender_R2/r2_rendertarget.cpp`](../xrRender_R2/r2_rendertarget.cpp.md) ·
[`../xrRender_R2/r2_rendertarget_phase_bloom.cpp`](../xrRender_R2/r2_rendertarget_phase_bloom.cpp.md) ·
[`../xrRender_R2/r2_rendertarget_phase_PP.cpp`](../xrRender_R2/r2_rendertarget_phase_PP.cpp.md) ·
[`../xrRender_R2/r3_rendertarget_phase_ssao.cpp`](../xrRender_R2/r3_rendertarget_phase_ssao.cpp.md) ·
[`../xrRenderGL/glSH_RT.cpp`](../xrRenderGL/glSH_RT.cpp.md) ·
[`../xrRenderGL/glHW.cpp`](../xrRenderGL/glHW.cpp.md) ·
[`../xrRender/dxEnvironmentRender.h`](../xrRender/dxEnvironmentRender.h.md) ·
[Seam: Graphics device](../../../SYSTEM-REQUIREMENTS.md#seam-graphics-device)
**Used by** — [`gl_rendertarget.h`](gl_rendertarget.h.md)
**Tier floor** — T1: framebuffer attachment rewrites, a multisample blit, and quads whose
vertex order encodes the API's window origin.

## Purpose

The R2 chapter stops at "combine" because the composition's *target juggling* is the one
part that differs between fillings. This is that part, for OpenGL.

The **effect order is shared design** and is written out once, in the sibling twin
[`../xrRenderPC_R4/r4_rendertarget_phase_combine.cpp`](../xrRenderPC_R4/r4_rendertarget_phase_combine.cpp.md):
occlusion, sky and clouds, the deferred resolve quad, the forward pass, volumetric
compositing, multisample resolve, bloom, the distortion mask, the final
antialias/motion-blur/depth-of-field quad, lens flares, the film post-process, exposure
swap. A rebuilder must keep that order regardless of API. This page records only what this
API forces to be different, and why.

Four things force a difference, and only one of them changes a pixel.

**1. The window origin is at the other corner.** This API puts the framebuffer's origin at
the bottom-left; the other filling's is at the top-left. Every full-screen quad that
carries explicit texture coordinates therefore samples upside down unless its vertices are
emitted in the opposite vertical order. In this file the final antialias quad is emitted
in that opposite order — the same four positions and the same seven coordinate sets, rows
one and two swapped and rows three and four swapped. It is the only difference between the
two files that changes an output pixel, and it is the kind of difference that is invisible
in a screenshot of a symmetric scene, which is why it is worth naming. A rebuild should
settle the convention once, at whatever builds full-screen quads, and never per call site.

**2. The clip-space depth range is different.** This API's native range runs minus one to
one; the other's runs zero to one, and the engine standardizes on zero to one. This file
escapes the consequence because its quads sit at a near-zero constant depth, but the
matrices that project into shadow-map texture space do not — every such matrix in the
renderer carries an extra half-scale and half-bias on its depth row for this API. See
[`../xrRender_R2/r3_rendertarget_draw_rain.cpp`](../xrRender_R2/r3_rendertarget_draw_rain.cpp.md)
and [`gl_rendertarget_accum_direct.cpp`](gl_rendertarget_accum_direct.cpp.md).

**3. Two facilities are not available and their branches are hollow.** The compute-shader
ambient occlusion the other filling runs at the top of the combine does not exist here: the
branch that selects it is present and empty, so requesting the highest occlusion quality
silently yields *no occlusion at all* rather than falling back to the screen-space pass.
And the multisample edge work has only its optimized form — the per-sample loop over
sample masks aborts outright, because the sample-addressing the program form needs is the
only one this backend was written for.

**4. A copy replaces a draw at the two places a draw was only moving pixels.** The
multisample resolve is a framebuffer blit rather than a device resolve call, and the
presentation of the finished image is a framebuffer copy rather than the textured quad the
other filling once used (see [`gl_rendertarget_phase_flip.cpp`](gl_rendertarget_phase_flip.cpp.md)).
Both matter to the caller: the blit is described below, and the presentation copy is why
nothing in this file ever binds the window's own buffer.

## State

```text
saved_view_projection : matrix     # the previous frame's full view-projection,
                                   #   the motion-blur reference; written at the
                                   #   end of every combine
```

**Invariants** — one slot, not one per command context: the combine runs once per frame on
the immediate context. Everything else this file touches is owned by the render-target set
declared in [`gl_rendertarget.h`](gl_rendertarget.h.md). The construction of the two blur
matrices from this slot is shared design and lives in the sibling twin.

## `phase_combine`

**Contract** — as the sibling's: reads the accumulator, the G-buffer, the occlusion scratch
and the exposure pool; writes a low-dynamic-range target that the film post-process copies
to the presentable surface; leaves the exposure pool swapped. The differences are listed
above and detailed below. Does not present.

### The multisample resolve is an in-framebuffer blit

**Contract** — when multisampling is on, both screen scratch targets are copied from their
multisample allocations to their single-sample twins, between the forward pass and bloom.

```text
FUNCTION resolve(source, destination) -> ()
  attach source      as colour slot 0 of the one framebuffer
  attach destination as colour slot 1
  read from slot 0, draw to slot 1
  assert the framebuffer is complete
  blit the whole rectangle, colour only, nearest filtering
```

**Invariants** — this leaves the single framebuffer's attachments pointing at the resolve
pair, not at whatever the combine had bound. Every path out of the resolve must rebind its
targets before drawing again, and does. A rebuild on an API with a dedicated resolve
operation has no such hazard and should not imitate this shape.

**Invariants** — nearest filtering, and identical source and destination rectangles. The
blit is a resolve, not a rescale; a filtered blit between different sizes would silently
turn the resolve into a downsample and lose the multisample geometry the antialias filter
is about to look for.

### Multisampling: what is actually drawn

**Contract** — the sibling describes a three-way shape: an interior draw with the
per-pixel program, then either one per-sample draw over edge pixels or one draw per sample.
This filling implements only the edge draw.

```text
IF multisampling is off
    one draw with the ordinary combine program, stencil admits covered pixels
ELSE
    select the per-sample combine program
    IF the optimized path is available
        one draw, stencil admits ONLY edge pixels (high stencil bit set)
    ELSE
        abort
    stencil off
```

**Notes** — **the interior draw is absent**, and its absence looks like a defect rather
than a decision: the per-sample program is selected before the branch that would have
chosen between interior and edge, and the only draw issued tests for the edge bit, so
non-edge pixels receive no resolve at all. The other filling issues both draws from the
same code. No comment or commit message in the source explains the divergence, so whether
multisampled output on this backend is known-broken or the program compensates is **not
recoverable** from the source alone. A rebuild should implement the sibling's three-way
shape and treat this as the bug it appears to be.

### Binding the exposure pool

**Contract** — the two exposure-pool entries are attached to the source and destination
sampler names before the pass and detached after, as the sibling does. Here a texture is
identified by its shape and its device name rather than by a view object, so the attach
carries the two-dimensional texture target alongside the name and the detach passes a zero
name. This is an accident of how the API names resources and survives a rebuild as nothing
at all.

## `phase_combine_volumetric`

**Contract** — identical to the sibling's: one full-screen quad compositing the volumetric
accumulator into the two scratch targets, colour channels only. Its quad is given directly
in clip space with texture coordinates already in the zero-to-one range, so the window
origin does not reach it.

## `phase_wallmarks`

**Contract** — identical to the sibling's: detaches the second and third colour targets,
binds albedo alone with the multisample depth, admits only pixels the G-buffer covered,
culls back faces and enables the three colour channels but not alpha. Sets no geometry and
issues no draw.

**Notes** — the detach is spelled as attaching a zero handle rather than a null view. Same
meaning, same requirement: position and normal must be off the framebuffer before a decal
is drawn, or the decal rewrites the geometry buffer.

## `hclip`

**Contract** — maps a pixel coordinate on an axis of given length onto the clip-space range
minus one to one. Not called: every live quad in this file is already written in clip
space, and the only uses are in the commented-out predecessor of the resolve quad.

**Notes** — it is recorded because its existence is the evidence for the convention this
file settled on. The predecessor built its quad in pixel coordinates with a half-pixel
offset and converted; the live version skips both steps. The half-pixel offset belonged to
a sampling rule neither of this chapter's APIs has, and a rebuild should carry neither the
helper nor the offset.
