# src/Layers/xrRender/du_box.h

> Declares the unit cube tables and their sizes.

**Needs** — [`du_box.cpp`](du_box.cpp.md)
**Used by** — [`D3DUtils.cpp`](D3DUtils.cpp.md) · [`du_box.cpp`](du_box.cpp.md)
**Tier floor** — T2: a declaration only.

## Purpose

Declares the tables defined in [`du_box.cpp`](du_box.cpp.md), and the four counts a caller needs to size its buffers: 8 vertices, 12 triangles, 12 lines, and 36 unindexed vertices. Callers rely on the counts as compile-time constants — they size stack arrays with them — so they belong with the declaration rather than being derived from the tables.

Exported units: `du_box_vertices`, `du_box_faces`, `du_box_lines`, `du_box_vertices2`.
