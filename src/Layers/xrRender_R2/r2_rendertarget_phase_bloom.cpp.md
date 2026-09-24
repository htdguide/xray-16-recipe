# src/Layers/xrRender_R2/r2_rendertarget_phase_bloom.cpp

> Downsamples the lit frame to a quarter-size bright-pass image and blurs it, either with
> a cheap four-tap kernel or with a separable fifteen-tap Gaussian.

**Needs** — [`r2_rendertarget.cpp`](r2_rendertarget.cpp.md) · [`r2_types.h`](r2_types.h.md) · [`r2_rendertarget_phase_luminance.cpp`](r2_rendertarget_phase_luminance.cpp.md) · [`xrRender/blenders/blender_bloom_build.h`](../xrRender/blenders/blender_bloom_build.h.md) · [`xrEngine/Environment.h`](../../xrEngine/Environment.h.md)
**Used by** — [`r4_rendertarget_phase_combine.cpp`](../xrRenderPC_R4/r4_rendertarget_phase_combine.cpp.md)
**Tier floor** — T1: it builds per-vertex tap offsets against a fixed target size.

## Purpose

Bloom is the glow around bright things. It is computed at a fixed 256×256 regardless of
screen resolution — low frequency by nature, so resolution buys nothing — and it doubles
as the source for exposure measurement, which is why the luminance reduction is called
from the middle of this pass rather than standing on its own.

Two filters ship. The fast one is four bilinear taps at a diagonal offset, run twice. The
slow one is a real separable Gaussian: fifteen taps horizontally, then fifteen vertically,
with weights computed on the processor each frame because the kernel radius is a live
setting.

## `phase_bloom`

**Contract** — reads the scene image, writes the finished bloom into the first bloom
target, and leaves that target bound. Calls the luminance reduction partway through.
Disables depth for the duration. When the menu's own post-process is active it clears the
result to black instead, so the menu is not bloomed by the scene behind it.

```text
FUNCTION phase_bloom()
  bind bloom target 1; depth off

  # --- bright-pass downsample: screen -> 256x256, four taps ------------------
  # the quad carries four coordinate sets, offset by one source texel in a 2x2
  # pattern, each scaled by the ratio of half the screen to the bloom size so the
  # taps stay one *screen* texel apart regardless of resolution
  threshold = the configured bloom threshold
  bloom_intensity = 0.9 * bloom_intensity + 0.1 * configured_speed * frame_delta
  constant: (threshold, threshold, threshold, bloom_intensity)
  draw with the build element

  # --- exposure --------------------------------------------------------------
  phase_luminance()          # reads this same 256x256 image

  # --- blur ------------------------------------------------------------------
  IF the fast filter is selected
      offsets = a diagonal 2x2 at the configured fast-kernel radius
      bloom 1 -> bloom 2 with element 3
      bloom 2 -> bloom 1 with element 4
  ELSE
      # horizontal: fifteen taps, centre plus seven each side, packed as eight
      # coordinate sets carrying a left and a right tap each
      weights = gauss_wave(radius, radius/3, scale)
      bloom 1 -> bloom 2 with element 1 and those weights
      # vertical: the same, with the radius corrected by the screen's aspect ratio
      weights = gauss_wave(radius * height/width, that/3, scale)
      bloom 2 -> bloom 1 with element 2 and those weights

  IF the menu's post-process is active THEN clear the result to black
  depth on
```

## `gauss_wave` — the kernel weights

**Contract** — produces eight weights covering a fifteen-tap symmetric kernel, as the sum
of two Gaussians of different radii. Pure; computed per frame because the radius is a
setting.

```text
FUNCTION gauss_k7(radius, magnitude) -> eight weights
  w[i] = exp( -i^2 / (2 * radius^2) )   for i in 0..7
  total = w[0] + 2 * sum(w[1..7])       # the kernel is symmetric
  RETURN w scaled by magnitude / total

FUNCTION gauss_wave(base_radius, detail_radius, magnitude) -> eight weights
  RETURN gauss_k7(base_radius, magnitude) + gauss_k7(detail_radius, magnitude)
```

**Notes** — summing a wide and a narrow Gaussian is what makes the glow look like a lens
rather than like a blur: a real lens flare has a tight core and a broad halo, and a single
Gaussian gives only one of them. The detail radius is fixed at a third of the base, so one
setting controls both.

The weights are *not* renormalized after the sum, so the combined kernel integrates to
twice the requested magnitude. That is absorbed by the magnitude setting being tuned
against the combined result; a rebuild that normalizes will need to halve the default.

**Notes on the vertical radius.** The vertical pass scales its radius by the ratio of
screen height to width. Because the bloom target is square (256×256) while the screen is
not, one bloom texel covers more screen distance horizontally than vertically; correcting
the radius is what keeps the glow circular on screen instead of elliptical.

**Notes on the intensity smoothing.** The bloom factor uses the same nine-to-one
exponential smoothing as the exposure adaptation, against a configured speed and the frame
delta. Both are settling times of roughly ten frames, and both exist so that a sudden
change in scene brightness ramps rather than pops.

The fast filter's two passes are not separable — both apply the same diagonal 2×2 — so it
is a box blur applied twice, not a Gaussian. It is visibly boxier and is there for weak
hardware.
