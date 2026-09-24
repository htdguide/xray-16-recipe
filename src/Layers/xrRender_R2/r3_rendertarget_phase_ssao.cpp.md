# src/Layers/xrRender_R2/r3_rendertarget_phase_ssao.cpp

> Computes screen-space ambient occlusion at half resolution, and the depth downsample it
> reads.

**Needs** — [`r2_rendertarget.cpp`](r2_rendertarget.cpp.md) · [`r2_types.h`](r2_types.h.md) · [`xrRender/blenders/blender_ssao.h`](../xrRender/blenders/blender_ssao.h.md)
**Used by** — [`r4_rendertarget_phase_combine.cpp`](../xrRenderPC_R4/r4_rendertarget_phase_combine.cpp.md) · [`r4_rendertarget_phase_hdao.cpp`](../xrRenderPC_R4/r4_rendertarget_phase_hdao.cpp.md)
**Tier floor** — T1: viewport manipulation and target binding around a full-screen draw.

## Purpose

Ambient occlusion darkens creases and contact points by sampling the neighbourhood of each
pixel's position and counting how much of the hemisphere is blocked. It runs at half
resolution into its own target, which the combine pass then reads. A companion pass
downsamples the position target into a single-channel depth image, because the sampling
kernel takes many taps and taking them from the fat position target would be bandwidth
bound.

Which of several occlusion algorithms runs is an option resolved in
[`r2.cpp`](r2.cpp.md); this file is the plain screen-space one and the shared downsample.
The horizon-based and high-definition variants are backend files.

## `phase_ambient_occlusion`

**Contract** — clears the occlusion target, binds it with no depth, sets the viewport to
half the screen, and draws one full-screen quad with the occlusion material. Restores the
full viewport and clears stencil state on the way out.

```text
FUNCTION phase_ambient_occlusion()
  clear the occlusion target to zero
  bind it; no depth target; stencil off
  # kernel scale: both the noise tiling and the sampling radius are corrected for
  # field of view against a reference of 67.5 degrees
  noise_tiling = 2      * tan(67.5 deg) / tan(field_of_view)
  kernel_size  = 150    * tan(67.5 deg) / tan(field_of_view)
  viewport = half the screen
  quad texture coordinates tile the jitter texture at one texel per half-res pixel
  material = the occlusion description, element 0
  constants: the inverse view matrix, the noise tiling, the kernel size,
             and (width, height, 1/width, 1/height) at half resolution
  draw
  restore the full viewport; stencil off
```

**Invariants** — the target has no depth attachment, so nothing rejects pixels; occlusion
is computed everywhere and gated at combine time instead. The viewport, not the target
size, is what halves the resolution, which means the target may be full size and only
half used — that is the case when the half-resolution data option is off.

**Notes** — the field-of-view correction on *both* the noise tiling and the kernel is the
load-bearing part. The kernel is expressed in world units at a reference field of view; at
a narrower field of view the same world distance covers more pixels, so the kernel must
grow in screen space or the occlusion visibly shrinks when the player aims down a scope.
The reference of 67.5 degrees is the game's default vertical field of view.

The multisampled per-sample variant is written and commented out: occlusion is computed
per pixel even under multisampling. That is a deliberate quality-for-cost trade, since
occlusion is low frequency and the edges are already resolved by the lighting passes.

## `phase_depth_downsample`

**Contract** — writes a reduced-precision depth image from the position target, at half or
full resolution depending on the option, for the occlusion kernel to sample.

```text
FUNCTION phase_depth_downsample()
  bind the half-depth target; no depth attachment; clear to zero
  IF half-resolution data is on THEN viewport = half the screen
  quad texture coordinates tile the jitter texture at one texel per output pixel
  material = the occlusion description, element 1
  constant: the inverse view matrix
  draw
  restore the full viewport
```

**Notes** — the target's format is chosen per vendor at construction — a 32-bit float on
one, a 16-bit float on the other — because 16-bit is enough precision for an occlusion
kernel's depth comparisons but one vendor's 16-bit float filtering was slower than its
32-bit. That is hardware trivia; the decision that survives is that the occlusion kernel
needs *a* separate narrow depth image, not the fat position target.

The source carries a struck-out attempt to do this with a device-level stretch-copy
instead of a draw, marked "don't do this". A copy cannot apply the view transform, and the
occlusion kernel wants linear eye-space depth rather than whatever the position target
holds; the draw is not a workaround, it is the operation.
