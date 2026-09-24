# src/Layers/xrRender/r__pixel_calculator.h

> Declares the offline per-direction coverage measurement tool.

**Needs** — [`r__pixel_calculator.cpp`](r__pixel_calculator.cpp.md) · [`Include/xrRender/RenderVisual.h`](../../Include/xrRender/RenderVisual.h.md)
**Used by** — [`r__pixel_calculator.cpp`](r__pixel_calculator.cpp.md) · [`xrRender_console.cpp`](xrRender_console.cpp.md)
**Tier floor** — T2: a declaration plus one small record.

## Purpose

Declares the tool implemented in [`r__pixel_calculator.cpp`](r__pixel_calculator.cpp.md).

## State

```text
RECORD Coverage
  ratio : list<int (8-bit)> of exactly 6   # +x, -x, +y, -y, +z, -z
```

Six eight-bit fill ratios, one per axis direction, in the same face order the cube-face light split uses. Eight bits because a coverage correction never needs more precision than one part in two hundred and fifty-six, and the record is meant to be stored per model in the level data.

## Exported units

- **`PixelCalculator`** (`r_pixel_calculator`) — owns a private render target and depth buffer; exposes `begin`, `calculate`, `end` and `run`. The measurement is described in the implementation twin.
