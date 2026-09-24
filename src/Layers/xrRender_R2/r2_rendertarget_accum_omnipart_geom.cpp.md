# src/Layers/xrRender_R2/r2_rendertarget_accum_omnipart_geom.cpp

> Uploads the sphere cap that bounds one face of a cube-shadowed point light.

**Needs** — [`r2_rendertarget.cpp`](r2_rendertarget.cpp.md) · [`xrRender/du_sphere_part.h`](../xrRender/du_sphere_part.h.md) · [Seam: Graphics device](../../../SYSTEM-REQUIREMENTS.md#seam-graphics-device)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T1: it maps device buffers and copies a fixed byte image into them.

## Purpose

A point light that casts shadows is decomposed into up to six sub-lights, one per face of
a cube, each with its own shadow rectangle in the atlas and each covering a ninety-degree
pyramid of the original sphere. The volume for one of those sub-lights is not a sphere and
not a cone but a *sphere cap* — the intersection of the sphere with the face's pyramid.
This file uploads that mesh, position-only, exactly as the point sphere is uploaded.

## `create_omnipart_volume`

**Contract** — creates and fills the sphere-cap volume's vertex and index buffers from the
constant table. Called once at target-set construction.

## `destroy_omnipart_volume`

**Contract** — releases both buffers.

**Notes** — using the cap rather than the whole sphere is what keeps a shadowed point light
affordable: each of the six sub-lights marks and shades only its own sixth of the sphere,
so the total rasterized area is the sphere's, not six times it.
