# src/Layers/xrRender_R2/r2_rendertarget_accum_point_geom.cpp

> Uploads the unit sphere that stands in for a point light's volume.

**Needs** — [`r2_rendertarget.cpp`](r2_rendertarget.cpp.md) · [`xrRender/du_sphere.h`](../xrRender/du_sphere.h.md) · [Seam: Graphics device](../../../SYSTEM-REQUIREMENTS.md#seam-graphics-device)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T1: it maps device buffers and copies a fixed byte image into them.

## Purpose

A point light is bounded by a sphere. The sphere is a compile-time constant table of
positions and triangle indices (chapter 18's debug-geometry set, reused here); this file
is only its upload. Positions are three floats and nothing else — no normal, no texture
coordinate — because the volume is never shaded, only rasterized to mark stencil and to
run the lighting program over the pixels inside it.

## `create_point_volume`

**Contract** — creates a vertex buffer of the sphere's positions and an index buffer of
its triangles, filling both by mapping and copying. Called once when the target set is
built. Fails if the device refuses the allocation.

```text
FUNCTION create_point_volume()
  vertex buffer  = sphere vertex count * 12 bytes   # three floats, position only
  index buffer   = sphere triangle count * 3 * 2 bytes
  map each, copy the constant table straight in, unmap with upload
```

## `destroy_point_volume`

**Contract** — releases both buffers.

**Notes** — the sphere's tessellation is fixed by the table and is a real decision: too
coarse and the volume clips light near the sphere's surface, too fine and the double
rasterization for the stencil bound costs more than it saves. The shipped mesh is on the
coarse side, which is why the lighting program's attenuation is evaluated against the
light's *analytic* radius rather than the mesh — the mesh only has to contain the sphere,
not match it.
