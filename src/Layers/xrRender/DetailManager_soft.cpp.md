# src/Layers/xrRender/DetailManager_soft.cpp

> The fallback grass draw path: transform every plant's vertices on the processor into the shared dynamic stream, in chunks small enough to keep the buffer from stalling.

**Needs** — [`DetailManager.h`](DetailManager.h.md) · [`R_DStreams.h`](R_DStreams.h.md) · [`R_Backend.h`](R_Backend.h.md) · [Seam: Graphics device](../../../SYSTEM-REQUIREMENTS.md#seam-graphics-device)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T1: the whole file is filling a mapped device buffer with an exact vertex layout.

## Purpose

One of two ways the grass reaches the screen. This one is for machines with no programmable vertex stage: every visible plant's vertices are transformed on the processor and written into the renderer's shared dynamic vertex stream. It has no wind — plants drawn this way never move — and it draws only the still batch.

**This path is dead on any machine built this century.** It survives because the oldest renderer generation can be forced into fixed-function operation. It is documented here because the *chunking* decision in it is the same decision the live path makes, and because it is the clearest statement of what the hardware path is avoiding.

## The vertex budget

```text
vertices_per_lock = 3000
```

**Invariants** — The per-map vertex budget is the file's only tuning number and it is what turns an unbounded instance list into a bounded number of buffer maps. It is a *soft* budget: the code divides the instance list into the smallest number of chunks whose per-chunk vertex count does not exceed it, then *distributes the instances evenly* across those chunks rather than filling each to the brim. The even distribution keeps the last chunk from being a stub, which matters because each chunk is a draw call and a one-instance draw costs as much in state as a hundred.

Three thousand vertices is roughly a 100 KB map on this vertex layout. There is no recoverable derivation; it is the size at which the dynamic stream's ring does not stall on the hardware of the era.

## `soft_load()` / `soft_unload()`

**Contract** — declares the geometry layout — position, packed colour, one texture coordinate pair — over the renderer's *shared* dynamic vertex and index streams. The path owns no buffers of its own; that is the point of it.

## `soft_render()`

**Contract** — draws every still-batch plant. Maps and unmaps the shared streams repeatedly. Called with back-face culling already off. Clears the visible lists as it consumes them.

```text
FUNCTION soft_render()
  FOR EACH model index O
    FOR EACH visible instance list IN visible[still][O]
      chunks = ceil(list.count * model.vertex_count / 3000)
      per_chunk = ceil(list.count / chunks)
      bind the model's material
      FOR EACH chunk
        map the shared vertex stream for (chunk_size * model.vertex_count) vertices
        map the shared index stream  for (chunk_size * model.index_count)  indices
        index_offset = 0
        FOR EACH instance IN the chunk
          build its transform: its yaw-and-position matrix with the rotation
            columns scaled by the instance's faded scale
          write its transformed vertices; write its indices shifted by index_offset
          index_offset = index_offset + model.vertex_count
        unmap both
        draw the chunk as one indexed triangle list
    clear visible[still][O]
```

**Invariants**

- The per-instance transform is built by scaling the rotation columns in place rather than composing a scale matrix — the plant's transform is always "uniform scale, then yaw, then translate", so the composition is three multiplies per row instead of a full matrix product. At tens of thousands of instances that is worth having, and it is a decision a rebuild reproduces rather than a spelling.
- Every vertex is written with an opaque white colour. The software path carries **no baked lighting** — the per-cell sun and sky values the decompressor computed are simply not used here. The plants are drawn unlit. That is the visible quality difference between the two paths and it was evidently acceptable on machines that needed this one.
- The index shift is the same two-at-a-time rebasing described in [`DetailModel.cpp`](DetailModel.cpp.md), inlined here rather than called. The duplication is real and buys nothing.
- The visible lists are cleared *by the draw*, not by the visibility pass. That coupling means the draw must run exactly once per visibility pass. The hardware path has the same coupling.

**Notes** — Writing the vertices here rather than calling the model's own `transfer` is another duplication: `transfer` does exactly this. The inline copy exists so the loop can hoist the material bind and the buffer map out of it, which `transfer`'s signature does not allow. A rebuild should keep one copy.
