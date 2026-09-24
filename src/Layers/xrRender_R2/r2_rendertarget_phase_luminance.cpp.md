# src/Layers/xrRender_R2/r2_rendertarget_phase_luminance.cpp

> Reduces the frame to one exposure value in three steps, and smooths it against the
> previous frame so the eye adapts rather than jumps.

**Needs** — [`r2_rendertarget.cpp`](r2_rendertarget.cpp.md) · [`r2_types.h`](r2_types.h.md) · [`xrRender/blenders/blender_luminance.h`](../xrRender/blenders/blender_luminance.h.md)
**Used by** — [`r2_rendertarget_phase_bloom.cpp`](r2_rendertarget_phase_bloom.cpp.md)
**Tier floor** — T1: a fixed reduction chain over device targets with hand-built sample
coordinates.

## Purpose

Tone mapping needs to know how bright the frame is. That number is produced by reducing
the already-downsampled bloom image to a single texel, in three steps, each one shrinking
by a factor of eight and averaging in the logarithmic domain — a log-average, not a mean,
because perceived brightness is logarithmic and a single specular highlight would
otherwise dominate an arithmetic mean.

The result lands in a one-by-one target that *survives the frame*, and is the only piece
of cross-frame state in the whole render-target set.

## `phase_luminance`

**Contract** — runs the three reduction steps and writes this frame's exposure into the
write half of the exposure pool. Reads the bloom target, not the accumulator: bloom has
already produced a 256×256 downsample of the scene and reusing it is free. Disables depth
and stencil for the duration and restores depth on exit. Called from inside the bloom pass.

```text
FUNCTION phase_luminance()
  stencil off; no culling; colour writes on; depth off

  # --- step 1: 256x256 -> 64x64, four taps per output texel ------------------
  bind the 64x64 target
  quad carries four texture coordinate pairs, offset by (0,0) (1,0) (0,1) (1,1)
      in source texels, all biased by half a texel
  material element 0; draw

  # --- step 2: 64x64 -> 8x8, sixteen taps per output texel -------------------
  bind the 8x8 target
  build sixteen sample offsets on an odd grid: x over 1,3,5,7 and y over 1,3,5,7,
      divided by the source size; the quad carries them as eight coordinate pairs,
      packed two samples per pair
  material element 1; draw

  # --- step 3: 8x8 -> 1x1, sixteen taps ---------------------------------------
  pool_slot = (frame number modulo the number of GPUs) * 2 + 1      # the write half
  bind that slot
  same sixteen-tap kernel, against a source size of 8
  adaptation = 0.9 * adaptation + 0.1 * frame_delta * adaptation_rate
  amount = tone mapping enabled ? the configured amount : 0
  tone_parameters = lerp( (1, 0, 1), (middle_grey, 1, low_luminance), amount )
  material element 2
  constant: (tone_parameters, adaptation)
  draw

  depth on
```

**Invariants** — the odd-numbered sample grid (1, 3, 5, 7) is what makes sixteen taps
cover a 64-texel region exactly: with bilinear filtering each tap at an odd half-texel
position averages a 2×2 block, so sixteen taps read all 64 source texels once. A grid on
even positions would double-count and miss.

**Notes on the adaptation rate.** The smoothing is two-stage and that is deliberate. The
*rate* itself is low-passed — nine parts old, one part new — before being handed to the
program, which then uses it to blend this frame's measurement against the previous frame's
stored exposure. Low-passing the rate rather than the value means a frame-rate spike
changes how fast the eye adapts for a few frames instead of causing a visible exposure
jolt. The constants nine-tenths and one-tenth are an exponential smoothing factor and are
the same pair used for the bloom intensity; they correspond to a time constant of roughly
ten frames.

**Notes on the tone-mapping parameters.** Three numbers reach the program: a middle-grey
target, a scale, and a low-luminance floor. When tone mapping is disabled the amount is
zero and the triple degenerates to (1, 0, 1), which the program's arithmetic turns into an
identity — so disabling tone mapping costs a lerp, not a branch. Interpolating toward the
neutral triple rather than switching also means the user's amount slider is continuous.

**Notes on the pool.** Two one-texel targets per GPU, addressed by frame parity, and
swapped after the combine pass reads them
(see [`r2_rendertarget.cpp`](r2_rendertarget.cpp.md) and the backend combine). Two are
needed because the program reads last frame's exposure while writing this frame's; the
per-GPU multiplication exists because alternate-frame multi-GPU rendering gives each GPU
its own copy of every resource, and a single shared pair would serialize them. On a single
GPU the modulo is always zero and the pool is just a pair.

The pool is cleared to a mid value at construction, not to zero, so the first frame after
a device reset is exposed plausibly instead of pitch black or blinding.
