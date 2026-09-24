# src/Layers/xrRender/r__pixel_calculator.cpp

> An offline tool that measures how many pixels each model actually covers from each of six directions, by rendering it into a private target and counting fragments with an occlusion query.

**Needs** — [`r__pixel_calculator.h`](r__pixel_calculator.h.md) · [`r__occlusion.h`](r__occlusion.h.md) · [`FBasicVisual.h`](FBasicVisual.h.md) · [`light.h`](light.h.md) · [`R_Backend.h`](R_Backend.h.md) · [`ResourceManager.h`](ResourceManager.h.md) · [Seam: Graphics device](../../../SYSTEM-REQUIREMENTS.md#seam-graphics-device)
**Used by** — [`r__pixel_calculator.h`](r__pixel_calculator.h.md)
**Tier floor** — T1: it creates its own render target and depth buffer, hijacks the frame bracket, and reads occlusion results synchronously.

## Purpose

The renderer's coverage estimate — bounding radius over squared distance — assumes an object roughly fills its bounding sphere. Many do not: a fence, a ladder, a chain-link gate have a large sphere and almost no pixels, and they survive the discard threshold long past the distance at which they contribute anything.

This tool measures the truth. For each model it renders the model from each of the six axis directions into a fixed-size target and counts the fragments that survive, yielding a per-direction fill ratio. The intent is that those ratios are baked into the model data and used to correct the estimate at run time.

It is a developer command, run from the console, not part of any frame. It is present on only one of the two backends; the other declares it unimplemented.

## State

```text
RECORD PixelCalculator
  target : RenderTarget    # a private square colour target
  depth  : DepthBuffer     # its matching depth buffer

RECORD Coverage
  ratio : list<int (8-bit)> of exactly 6   # per axis direction, the fill fraction as 0..255
```

The target is square with a side of one thousand and twenty-four. The size decides the measurement's resolution: a feature that covers less than one part in a million of the target rounds to zero. Nothing derives the number.

## `begin` / `end`

**Contract** — Create the private target and depth buffer, bind them, and open a frame; then, at the end, close the frame, restore the renderer's own base target and depth buffer, and release the private ones.

Opening a frame here is the awkward part: the tool runs from the console, outside the frame loop, and must bracket its own drawing. A rebuild that separates "the frame loop's frame" from "a render pass" does not need this.

## `calculate`

**Contract** — Measure one visual. Returns the six fill ratios. Blocks: twelve draws and six synchronous occlusion reads. Logs each ratio.

```text
FUNCTION calculate(visual) -> Coverage
  target_area = target_side^2

  FOR EACH face IN the six axis directions
    # The SAME six directions and up-vectors the cube-face light split uses
    # (see light.cpp). Reusing the table is what makes the six measurements
    # correspond to the six faces a point light is split into.
    view = camera looking along the face's axis from a hundred metres back,
           with the face's up vector
    box  = the visual's bounding box transformed into that view

    # An orthographic projection fitted EXACTLY to the transformed box, so the
    # object fills the target however large or small it is. That is what makes
    # the result a shape measurement rather than a size measurement.
    projection = orthographic_fitted_to(box)

    set transforms (world = identity, view, projection)

    # Draw twice. The first draw is thrown away: it establishes the depth
    # buffer so that self-occluding geometry is counted once. The second is
    # wrapped in the query and counts only the fragments that are actually the
    # nearest surface.
    clear the depth buffer
    bind the visual's material and draw it
    begin query ; draw it again ; end query

  FOR EACH face
    fragments = read query
    ratio     = clamp(fragments / target_area, 0, 1)
    result[face] = round(ratio * 255)
  RETURN result
```

**Invariants** — The depth buffer is cleared between faces but the colour target is not, because nothing reads the colour; only the fragment count matters. The two-draw scheme is essential: without the first draw, a model with overlapping surfaces would count every layer and report more than full coverage.

## `run`

**Contract** — Measure every mesh-bearing visual the renderer currently holds, logging the results. Skips visuals that are not meshes — hierarchies, skeletons, imposters — because the measurement is only meaningful for a leaf with geometry.

**Notes** — The results are only logged. Nothing writes them back into the model data and nothing at run time consumes them. The pipeline this tool was the front half of was never completed; the per-direction coverage field it produces exists in the format as a place to put them. A rebuild should either finish the loop — bake the ratios and multiply the coverage estimate by the ratio for the face most nearly facing the camera — or drop the tool.
