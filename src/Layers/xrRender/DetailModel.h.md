# src/Layers/xrRender/DetailModel.h

> Declares the concrete detail model: loads one from the level file, optimizes its triangle order, and stamps instances of it.

**Needs** — [`IRenderDetailModel.h`](IRenderDetailModel.h.md)
**Used by** — [`DetailManager.h`](DetailManager.h.md) · [`DetailModel.cpp`](DetailModel.cpp.md) · [`IRenderDetailModel.h`](IRenderDetailModel.h.md)
**Tier floor** — T1: it is the implementation of a device-facing interface.

## Purpose

Declares the surface implemented in [`DetailModel.cpp`](DetailModel.cpp.md): the one concrete filling of the detail-model interface described in [`IRenderDetailModel.h`](IRenderDetailModel.h.md).

## Exported units

- **`load(reader)`** — reads one model out of the level's detail file: its material, its parameters, its vertices and indices; then computes its bounds.
- **`optimize()`** — reorders triangles and permutes vertices for the hardware vertex cache, if that helps.
- **`unload()`** — releases geometry and material.
- **`transfer(...)`** — the two stamping operations the interface demands.

**Notes** — The split into an interface and this single implementation buys nothing at run time; there is exactly one filling. It exists because the authoring tools link a *different* filling with an editor-side material. A rebuild that does not build the tools can collapse the two files into one.
