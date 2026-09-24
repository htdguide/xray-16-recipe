# src/xrEngine/xrImage_Resampler.cpp

> Separable, filtered image rescaling — the only general-purpose resampler in the engine, used to squeeze a widescreen screenshot into a square thumbnail.

**Needs** — [`xrImage_Resampler.h`](xrImage_Resampler.h.md)
**Used by** — [`xrImage_Resampler.h`](xrImage_Resampler.h.md)
**Tier floor** — T1 as written, because it works on raw 32-bit pixel words with an explicit scanline stride. The *algorithm* is T3; nothing about filtered resampling needs manual memory.

## Purpose

An image has to be resized with better quality than nearest-neighbour: the save-game
thumbnail is the screen, non-square, scaled to a square. This is the classic separable
filtered-rescale — a horizontal pass into a scratch image, then a vertical pass into the
destination — with a menu of seven reconstruction filters.

It is an imported, largely unmodified piece of 1991 public-domain code, and the recipe
treats it that way: what matters is the algorithm and the filter definitions, not the
shape of the source.

## State

Stateless between calls. Within a call:

```text
RECORD ImageView
  width, height : int
  pixels        : bytes         # 32-bit words, four channels
  stride        : int           # in pixels, not bytes, despite the name

RECORD Contribution
  source_index : int
  weight       : real

RECORD ContributionList
  entries : list<Contribution>  # one list per destination row or column
```

**Invariant** — a destination sample's weights are built once per output coordinate and
reused for every row (or column) of the pass. That is the whole reason for the two-pass
separable structure: an N-by-N filter becomes two N-wide ones, and the weights are shared
across the perpendicular axis.

## The filters

Seven reconstruction kernels, each a function of distance and a *support radius* outside
which it is zero:

| Name | Support | Character |
|---|---|---|
| default (smoothstep) | 1 | `2t³ - 3t² + 1`; the same cubic ease used by the noise generator |
| box | 0.5 | nearest-neighbour; a pixel contributes to exactly one output |
| triangle | 1 | linear interpolation |
| bell | 1.5 | a box convolved with itself three times — a quadratic B-spline |
| B-spline | 2 | four box convolutions — a cubic B-spline; blurry and artefact-free |
| Lanczos-3 | 3 | windowed sinc; sharpest, and the only one that can ring |
| Mitchell | 2 | the Mitchell–Netravali cubic with both parameters at one third |

**Notes** — the Mitchell parameters at (1/3, 1/3) are the values Mitchell and Netravali
themselves recommended as the best trade-off between blurring and ringing. They are written
as named constants that are never varied; a rebuild may expose them, but the recommended
pair is the reason those two numbers appear.

The support radius is not cosmetic: it decides how many source pixels each output sample
touches, and therefore the cost. Lanczos-3 is six times the work of a triangle filter.

## `imf_Process`

**Contract** — rescales a 32-bit four-channel image from one size to another using a named
filter. Source and destination buffers are supplied by the caller and must not overlap;
both must be at least two pixels in each dimension. Allocates one intermediate image and
the weight tables, and frees them. Does not blocked or thread-hop. Channel order is
whatever the caller's is — the four channels are treated identically apart from one
rounding quirk, below.

```text
FUNCTION resample(dst, dst_w, dst_h, src, src_w, src_h, filter_kind)
  REQUIRE both images at least 2 by 2
  (kernel, support) = filter_kind
  scratch = new image (dst_w by src_h)          # horizontally scaled, vertically original

  weights_x = build_weights(dst_w, src_w, kernel, support)
  FOR EACH source row r
    row = src row r
    FOR EACH destination column c
      accumulate per channel: sum over weights_x[c] of weight * row[source_index]
      scratch[c, r] = clamp each channel to 0..255

  weights_y = build_weights(dst_h, src_h, kernel, support)
  FOR EACH destination column c
    column = scratch column c
    FOR EACH destination row r
      accumulate per channel: sum over weights_y[r] of weight * column[source_index]
      dst[c, r] = clamp each channel to 0..255
```

### Building the weights

```text
FUNCTION build_weights(dst_size, src_size, kernel, support) -> list<ContributionList>
  scale = dst_size / src_size
  IF scale < 1 THEN                    # minifying: widen the kernel to average, not alias
    radius = support / scale
    step   = 1 / scale
  ELSE                                 # magnifying: the kernel keeps its natural width
    radius = support
    step   = 1

  FOR EACH destination index i
    centre = i / scale                 # position in source space
    FOR j FROM ceil(centre - radius) TO floor(centre + radius)
      weight = kernel((centre - j) / step) / step
      source = reflect j into 0 .. src_size-1
      record (source, weight)
```

**Invariants** — two decisions here are load-bearing and easy to get wrong.

**Minification widens the kernel.** When shrinking, each output pixel must average a whole
neighbourhood of the source; a fixed-width kernel would sample sparsely and alias. The
radius is divided by the scale and the weights are divided by the same amount to keep the
sum at one. When magnifying, the kernel keeps its natural width — widening it there would
only blur.

**Out-of-range source indices are reflected, not clamped.** An index below zero becomes its
mirror about zero; an index past the end becomes its mirror about the last pixel. Reflection
is the right edge rule for a resampler because clamping over-weights the edge pixel and
produces a visible band. There is then a final clamp as a guard, which catches the case
where the reflected index is *still* out of range — possible when the kernel is wider than
the image.

**Notes** — the weights are not renormalised after construction. For kernels whose
continuous integral is one, the discrete sum is close enough; for Lanczos-3, which
overshoots, this is the source of the ringing, and it is inherent to the filter rather than
a defect here.

The alpha channel is rounded with an extra half added before the clamp, while the colour
channels are not. That asymmetry has **no discoverable reason** — it is not a gamma
correction and it is not a precision concern. A rebuild should round all four the same way.

## Notes on the original's shape

Every allocation in the function is wrapped in an exception handler that logs a short tag
and *continues*, leaving a null buffer that the next block dereferences. This is not error
handling; it is a diagnostic left in by hand. A rebuild should fail the call on an
allocation failure and return a result, and should note that the failure mode is not
reachable in practice because the buffers are small.

The intermediate image is allocated per call. Since the only caller resizes one screenshot
at save time, that is fine; a rebuild resampling in a loop should hoist it.
