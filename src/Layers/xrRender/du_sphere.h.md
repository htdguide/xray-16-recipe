# src/Layers/xrRender/du_sphere.h

> Declares the unit sphere's two meshes and their sizes.

**Needs** — [`du_sphere.cpp`](du_sphere.cpp.md)
**Used by** — [`D3DUtils.cpp`](D3DUtils.cpp.md) · [`du_sphere.cpp`](du_sphere.cpp.md) · [`r2_rendertarget_accum_point_geom.cpp`](../xrRender_R2/r2_rendertarget_accum_point_geom.cpp.md)
**Tier floor** — T2: a declaration only.

## Purpose

Declares the tables defined in [`du_sphere.cpp`](du_sphere.cpp.md), and the four counts a caller sizes its buffers with: 92 vertices and 180 triangles for the solid form, 60 vertices and 60 segments for the wire form.

Exported units: `du_sphere_vertices`, `du_sphere_faces`, `du_sphere_verticesl`, `du_sphere_lines`.
