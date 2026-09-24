# src/Layers/xrRender/R_Backend.cpp

> Builds the shared quad index buffer — the one immutable index pattern every sprite, particle and screen quad in the engine draws through.

**Needs** — [`R_Backend.h`](R_Backend.h.md) · [`BufferUtils.h`](BufferUtils.h.md) · [Seam: Graphics device](../../../SYSTEM-REQUIREMENTS.md#seam-graphics-device)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T1: it writes a 16-bit index array into a mapped device buffer.

## Purpose

Quad-shaped geometry — particles, decals, sprites, screen-space rectangles — is generated in bulk every frame. Its *vertices* differ per frame and go into the dynamic vertex ring; its *indices* never differ at all, because the *n*-th quad always uses vertices `4n .. 4n+3` in the same two triangles. So the indices are built once, at device creation, into an immutable buffer, and every quad batch draws with that buffer and a base vertex.

That is the entire file, and it is a separate file mostly by accident of the backend's own split. A rebuild may fold it into the backend's device-creation path.

## State

Stateless; it fills a buffer owned by the renderer.

## `CreateQuadIB`

**Contract** — Allocates an immutable, non-readable 16-bit index buffer holding the two-triangle pattern for **4096 quads** and fills it. Called once at device creation and again after a device reset. Blocks on the device's map and upload; allocates the buffer.

```text
FUNCTION create_quad_index_buffer()
  quad_count  = 4096
  index_count = quad_count * 6          # two triangles per quad

  buffer = create_index_buffer(index_count * 2 bytes, immutable, no readback)
  indices = map(buffer)

  vertex = 0
  write  = 0
  FOR i FROM 0 TO quad_count - 1
    indices[write + 0] = vertex + 0     # first triangle
    indices[write + 1] = vertex + 1
    indices[write + 2] = vertex + 2
    indices[write + 3] = vertex + 3     # second triangle, wound to match
    indices[write + 4] = vertex + 2
    indices[write + 5] = vertex + 1
    write  = write + 6
    vertex = vertex + 4

  unmap(buffer, upload = true)
```

**Invariants**

- **The quad's vertex order is the frozen part.** The pattern `(0,1,2)` and `(3,2,1)` says a quad's four vertices arrive in the order top-left, bottom-left, top-right, bottom-right — the two triangles share the diagonal from vertex 1 to vertex 2, and the second triangle's reversed listing is what keeps both triangles wound the same way. Every generator in the engine that writes quad vertices writes them in that order, and there is no marker anywhere saying so. A rebuild that emits `(0,1,2)` and `(1,2,3)` gets one of the two triangles back-facing and half of every particle disappears under back-face culling.
- 4096 quads is the **batch ceiling**, not a total: a generator that has more than 4096 quads to draw must split into several draws. The number is bounded above by the 16-bit index width — `4096 × 4 = 16384` vertices, comfortably inside 65536 — and chosen below that ceiling because the vertex ring, not the index buffer, is what a batch actually runs out of. A rebuild is free to raise it up to 16384 quads; nothing in the shipped data depends on the value.

**Notes** — The buffer is created without readback: it is written once from the processor and only ever read by the device. This matters on the device generation in question, where a readable buffer is placed in slower memory.
