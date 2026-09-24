# src/Common/NvMender2003/NVMeshMender.h

> Declares the mesh tangent-basis generator implemented in [`NVMeshMender.cpp`](NVMeshMender.cpp.md).

**Needs** — [`NVMeshMender.cpp`](NVMeshMender.cpp.md) · [`../d3d9compat.hpp`](../d3d9compat.hpp.md)
**Used by** — [`NVMeshMender.cpp`](NVMeshMender.cpp.md) · [`convert.h`](convert.h.md) · [`mender_input_output.h`](mender_input_output.h.md)
**Tier floor** — T2: a declaration of a batch geometry operation. Nothing in the surface is device- or format-facing.

## Purpose

Declares the surface implemented in [`NVMeshMender.cpp`](NVMeshMender.cpp.md), where the
contracts and the algorithm live. It also carries the vendor's original usage notes, which
are the best statement anywhere of what each tuning parameter means; those are folded into
the implementation twin.

## Exported units

- **`Vertex`** — the generator's own vertex record: position, normal, a two-component
  texture coordinate, tangent and binormal. The caller fills position and texture
  coordinate, and the normal too if it is not asking for normals to be computed.
- **`Mend`** — the single entry point. Takes vertices, indices and an empty
  new-to-old mapping, plus three crease thresholds, an area-weighting factor and three
  mode choices; rewrites all three inputs in place.
- **`NormalCalcOption`** — compute normals, or trust the ones supplied.
- **`ExistingSplitOption`** — decide triangle adjacency by vertex position, or by index.
- **`CylindricalFixOption`** — leave texture coordinates alone, or repair wrap-around
  seams before processing.

Everything else the class declares is internal to the algorithm and is covered in the
implementation twin.
