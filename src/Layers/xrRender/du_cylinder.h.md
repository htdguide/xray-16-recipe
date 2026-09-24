# src/Layers/xrRender/du_cylinder.h

> Declares the unit cylinder tables and their sizes.

**Needs** — [`du_cylinder.cpp`](du_cylinder.cpp.md)
**Used by** — [`D3DUtils.cpp`](D3DUtils.cpp.md) · [`du_cylinder.cpp`](du_cylinder.cpp.md)
**Tier floor** — T2: a declaration only.

## Purpose

Declares the tables defined in [`du_cylinder.cpp`](du_cylinder.cpp.md), and the three counts a caller sizes its buffers with: 26 vertices, 48 triangles, 30 lines. The line count carries a note that the full set of edges would be 36; six axis edges are deliberately not drawn.

Exported units: `du_cylinder_vertices`, `du_cylinder_faces`, `du_cylinder_lines`.
