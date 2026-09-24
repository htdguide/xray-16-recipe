# src/xrCore/vector.h

> An aggregation header: the whole math vocabulary in one include, under a packing directive that applies to all of it.

**Needs** — [`math_constants.h`](math_constants.h.md) · [`xr_types.h`](xr_types.h.md) · [`_std_extensions.h`](_std_extensions.h.md) · [`_vector2.h`](_vector2.h.md) · [`_vector3d.h`](_vector3d.h.md) · [`_vector4.h`](_vector4.h.md) · [`_sphere.h`](_sphere.h.md) · [`_matrix.h`](_matrix.h.md) · [`_matrix33.h`](_matrix33.h.md) · [`_quaternion.h`](_quaternion.h.md) · [`_rect.h`](_rect.h.md) · [`_fbox.h`](_fbox.h.md) · [`_fbox2.h`](_fbox2.h.md) · [`_obb.h`](_obb.h.md) · [`_cylinder.h`](_cylinder.h.md) · [`_plane.h`](_plane.h.md) · [`_plane2.h`](_plane2.h.md) · [`_color.h`](_color.h.md) · [`_compressed_normal.h`](_compressed_normal.h.md) · [`_random.h`](_random.h.md) · [`_flags.h`](_flags.h.md) · [`_math.h`](_math.h.md) · [`_bitwise.h`](_bitwise.h.md) · [`dump_string.h`](dump_string.h.md) · [`../xrCommon/math_funcs.h`](../xrCommon/math_funcs.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T1: it wraps every math type in a one-byte packing directive, which fixes their layouts.

## Purpose

One include for the whole math vocabulary: vectors of two, three and four components, 3x3 and 4x4 matrices, quaternions, rectangles, axis-aligned and oriented boxes, spheres, cylinders, planes, colours, compressed normals, the random generator and the bit-flag holder.

Its one substantive act is that **every type it includes is declared inside a one-byte packing region.** That is what guarantees the math types have no padding and can be read directly out of vertex buffers, level files and physics structures. A rebuild that does not pull them in through a single point must apply the same guarantee at each one.

The source carries its own verdict: the header is a hog, and it is the only place a handful of small helpers are defined, so it cannot simply be deleted. A rebuild should place those helpers where they belong and let consumers include what they use.

## Exported units

None of its own beyond the packing region.
