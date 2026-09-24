# src/Layers/xrRenderPC_R4/r4_rendertarget_phase_hdao.cpp

> The high-definition ambient-occlusion pass: the one place in the frame graph that runs as a compute dispatch over tiles rather than as a rasterized full-screen draw.

**Needs** — [`r4_rendertarget.h`](r4_rendertarget.h.md) · [`../xrRender_R2/r3_rendertarget_phase_ssao.cpp`](../xrRender_R2/r3_rendertarget_phase_ssao.cpp.md) · [`../xrRenderDX11/dx11R_Backend_Runtime.h`](../xrRenderDX11/dx11R_Backend_Runtime.h.md) · [Seam: Graphics device](../../../SYSTEM-REQUIREMENTS.md#seam-graphics-device)
**Used by** — [`r4_rendertarget_phase_combine.cpp`](r4_rendertarget_phase_combine.cpp.md)
**Tier floor** — T1: a compute dispatch with an explicitly written output view and an explicit unbinding of every input afterwards.

## Purpose

Ambient occlusion has three implementations in the engine, chosen by a quality setting: the
plain screen-space one and the horizon-based one are full-screen draws in the shared layer;
this is the third, and it exists only in this filling because it is the only pass in the
whole frame graph that wants a **compute dispatch writing to an arbitrary output image**
rather than a rasterized quad.

The reason it wants one is tiling. The occlusion kernel reads a neighbourhood around every
pixel, so neighbouring pixels read overlapping data. A compute dispatch can load one tile
of depth into fast shared storage once and have every pixel in that tile sample it, which a
pixel program cannot. That is the entire justification for the pass existing separately,
and the only thing a rebuilder needs to carry across.

## `phase_hdao`

**Contract** — runs the occlusion kernel over the whole bound output size and leaves the
result in the occlusion scratch target. Does nothing when ambient occlusion is disabled.
Unbinds every colour target before dispatching and every input and the output afterwards.
Blocks nothing, allocates nothing.

```text
FUNCTION phase_hdao()
  IF ambient occlusion is off THEN RETURN

  select the occlusion material's single pass
  apply its state, its program, its constants and its input textures

  unbind all colour targets            # nothing rasterizes here
  bind the occlusion scratch target as the writable output image

  # tiles overlap so that each tile's border pixels have the neighbourhood
  # their kernel needs without reading across a tile boundary
  tile_side    = 56
  tile_overlap = 12
  stride       = tile_side - 2 * tile_overlap     # = 32

  dispatch ceil(bound_width / stride) by ceil(bound_height / stride) tiles

  unbind the writable output image
  unbind every input texture slot
```

**Invariants**

- **The dispatch is sized by the stride, not the tile.** Each tile loads 56 pixels of data
  and produces the 32 in its middle; the 12-pixel margin on each side is the kernel's
  radius. Sizing the grid by 56 would leave gaps of unwritten pixels. This relationship —
  *tile − 2·overlap = what a tile produces* — is the load-bearing arithmetic on the page.
- **Colour targets must be unbound before the dispatch.** The occlusion scratch is bound as
  a writable image, and the device forbids the same allocation being reachable as both a
  colour target and a writable image; unbinding all four slots unconditionally is cheaper
  than working out which one it is.
- **Every input slot is cleared afterwards.** The occlusion result is sampled by the combine
  pass moments later; leaving it bound as an input to the compute stage would hold a hazard
  the device resolves by stalling.

## Notes

The pass runs on the immediate command context explicitly, not on whichever command list is
current. Occlusion is computed once for the main view only — the sun's and the rain's walks
have no use for it — so there is nothing to parameterize.

The overlap of 12 is the kernel's radius in pixels and is fixed; unlike the screen-space
occlusion pass, this one does **not** scale its kernel with field of view. That is an
inconsistency between the three occlusion implementations rather than a decision, and a
rebuild should apply the same field-of-view correction here that
[`r3_rendertarget_phase_ssao.cpp`](../xrRender_R2/r3_rendertarget_phase_ssao.cpp.md)
applies.

Whether this pass or one of the two draw-based ones runs is decided in
[`r4_rendertarget_phase_combine.cpp`](r4_rendertarget_phase_combine.cpp.md), on whether the
compute program compiled at all — a device that could not compile it falls back silently.
