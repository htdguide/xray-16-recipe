# src/Layers/xrRender_R2/r2_rendertarget_accum_spot_geom.cpp

> Uploads the cone that bounds a spot light, and builds the stack of parallel slices that
> light shafts are marched through.

**Needs** — [`r2_rendertarget.cpp`](r2_rendertarget.cpp.md) · [`xrRender/du_cone.h`](../xrRender/du_cone.h.md) · [Seam: Graphics device](../../../SYSTEM-REQUIREMENTS.md#seam-graphics-device)
**Used by** — [`r3_rendertarget_accum_spot.cpp`](r3_rendertarget_accum_spot.cpp.md)
**Tier floor** — T1: it maps device buffers and writes vertex and index images into them.

## Purpose

Two unrelated meshes that happen to share a file because both belong to spot lights. The
first is the cone, a constant table uploaded exactly like the point light's sphere. The
second is generated: a fixed number of unit quads stacked along one axis, which is the
geometry a volumetric spot light is drawn as.

## `create_spot_volume` / `destroy_spot_volume`

**Contract** — creates the cone's position-only vertex buffer and its index buffer from
the constant table, and releases them. Called once at target-set construction.

## `create_volumetric_slices` / `destroy_volumetric_slices`

**Contract** — generates a stack of unit quads and their indices. The stack is a fixed
count — one hundred — and its depth coordinate runs from zero to one inclusive.

```text
FUNCTION create_volumetric_slices()
  step = 1 / (slice_count - 1)
  t = 0
  FOR i IN 0 .. slice_count-1
      four vertices at (0,0,t) (0,1,t) (1,0,t) (1,1,t)
      t = t + step
  FOR i IN 0 .. slice_count-1
      two triangles over that slice's four vertices
```

**Invariants** — the slice count must be at least two, or the step is undefined. The
vertices are in a *unit* box, not in the light's space: the program transforms them by the
light's camera-space bounding box, which is why the quads carry no size of their own.

**Notes** — a hundred slices is the mesh's fixed capacity, not the number drawn. The draw
selects a prefix of the stack according to the light's quality setting, with a floor of
ten slices, and compensates for the reduced count by *scaling the light's colour up* and
stretching the box's far bound — fewer, thicker slices carrying proportionally more energy
each. That is the whole quality knob for light shafts, and it is in
[`r3_rendertarget_accum_spot.cpp`](r3_rendertarget_accum_spot.cpp.md).

Marching with camera-facing slices rather than ray-marching in the pixel program is a
choice of its era: it turns an unbounded loop into a fixed number of blended quads, which
the hardware of the time could do and a loop it could not.
