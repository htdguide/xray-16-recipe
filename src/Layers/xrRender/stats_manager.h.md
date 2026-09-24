# src/Layers/xrRender/stats_manager.h

> Declares the device-memory accounting totals and the per-pixel size helper.

**Needs** — [`stats_manager.cpp`](stats_manager.cpp.md) · [Seam: Graphics device](../../../SYSTEM-REQUIREMENTS.md#seam-graphics-device)
**Used by** — [`stats_manager.cpp`](stats_manager.cpp.md) · [`dx11HW.h`](../xrRenderDX11/dx11HW.h.md)
**Tier floor** — T2: a declaration plus one enumeration.

## Purpose

Declares the type implemented in [`stats_manager.cpp`](stats_manager.cpp.md).

## The purpose axis

```text
ENUM Purpose { vertex_buffer, index_buffer, render_target }
```

Three categories, and the choice of three is the whole classification decision: these are the resources the renderer allocates *itself*, in bulk, at run time. Textures are allocated by the resource manager and accounted separately; shader blobs and state objects are too small to matter. A rebuild adding, say, a structured-buffer category adds an entry here and the grid widens.

## Exported units

- **`MemoryStats`** (`stats_manager`) — the totals grid, publicly readable so the console command can format it; the paired increment and decrement entry points for each of the three resource kinds; and the raw size-and-pool forms for callers that have already computed a size.
- **`bytes_per_pixel`** — the per-format pixel size, in two overloads, one per backend's format enumeration.

**Notes** — The totals grid is public rather than accessed through a query. That is how the console's memory report reads it, and it is the only reader; a rebuild should expose a read-only view instead.
