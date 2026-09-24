# src/Layers/xrRender_R2/r3_rendertarget_mark_msaa_edges.cpp

> Sets the high stencil bit on every pixel whose multisamples disagree, so later lighting
> passes can run once per pixel in the interior and once per sample only on edges.

**Needs** — [`r2_rendertarget.cpp`](r2_rendertarget.cpp.md) · [`xrRender/blenders/dx11MSAABlender.h`](../xrRender/blenders/dx11MSAABlender.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T1: stencil state, a full-screen draw, and a multisampled depth target.

## Purpose

Deferred shading and multisampling do not combine naturally: the G-buffer has one sample
per subsample, so lighting would have to run per subsample everywhere, multiplying its
cost by the sample count. The escape is that only *edge* pixels actually differ between
their samples. This pass finds them once and records the answer in the stencil, and every
subsequent lighting pass then runs twice — a cheap per-pixel pass restricted to the
interior, and a per-sample pass restricted to the marked edges.

## `mark_msaa_edges`

**Contract** — draws one full-screen quad with the edge-detection material against the
multisampled depth target, writing the high stencil bit wherever the material's test
passes. Colour writes off, depth always passes and is not written, no culling. Leaves
colour writes enabled.

```text
FUNCTION mark_msaa_edges()
  fill a clip-space quad with texture coordinates
  bind only the multisampled depth target
  material = the edge-detection description
  stencil: always pass, write 0x80 under mask 0x80
  colour writes off; depth always, no write; no culling
  draw
```

**Invariants** — the write mask is the high bit alone, so the marking cannot disturb the
coverage bit or a light marker already stored. This is the same constraint that caps the
light-marker counter at 127 under multisampling rather than 255 (see
[`r2_rendertarget.cpp`](r2_rendertarget.cpp.md)).

**Notes** — the test itself is in the material's program, not here; it compares the
samples of the G-buffer at that pixel and reports disagreement. What this file fixes is
the *protocol*: the high bit means "edge", and every accumulation in the chapter reads it
that way. Three stencil comparisons recur throughout as a result — `== marker` for the
interior, `== marker | 0x80` for the edges, and `<= marker` for the paths that do not care.

Two backends differ here only in whether a colour attachment must accompany the depth
bind; the pass is otherwise identical.
