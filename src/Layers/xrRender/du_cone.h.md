# src/Layers/xrRender/du_cone.h

> Declares the unit cone tables and their sizes.

**Needs** — [`du_cone.cpp`](du_cone.cpp.md)
**Used by** — [`D3DUtils.cpp`](D3DUtils.cpp.md) · [`du_cone.cpp`](du_cone.cpp.md) · [`r2_rendertarget_accum_spot_geom.cpp`](../xrRender_R2/r2_rendertarget_accum_spot_geom.cpp.md) · [`r3_rendertarget_accum_spot.cpp`](../xrRender_R2/r3_rendertarget_accum_spot.cpp.md)
**Tier floor** — T2: a declaration only.

## Purpose

Declares the tables defined in [`du_cone.cpp`](du_cone.cpp.md), and the three counts a caller sizes its buffers with: 18 vertices, 32 triangles, 24 lines.

Exported units: `du_cone_vertices`, `du_cone_faces`, `du_cone_lines`.
