# src/Layers/xrRender/du_sphere_part.h

> Declares the sphere-wedge tables and their sizes.

**Needs** — [`du_sphere_part.cpp`](du_sphere_part.cpp.md)
**Used by** — [`D3DUtils.cpp`](D3DUtils.cpp.md) · [`du_sphere_part.cpp`](du_sphere_part.cpp.md) · [`r2_rendertarget_accum_omnipart_geom.cpp`](../xrRender_R2/r2_rendertarget_accum_omnipart_geom.cpp.md)
**Tier floor** — T2: a declaration only.

## Purpose

Declares the tables defined in [`du_sphere_part.cpp`](du_sphere_part.cpp.md), and the three counts a caller sizes its buffers with: 82 vertices, 160 triangles, 176 line segments.

Exported units: `du_sphere_part_vertices`, `du_sphere_part_faces`, `du_sphere_part_lines`.
