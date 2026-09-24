# src/Layers/xrRender/SkeletonXSkinXW.h

> The software skinning entry points — one per influence count — which the mesh dispatches to and which two implementations fill in.

**Needs** — [`SkeletonXVertRender.h`](SkeletonXVertRender.h.md) · [`SkeletonXSkinXW_CPP.cpp`](SkeletonXSkinXW_CPP.cpp.md) · [`SkeletonXSkinXW_SSE.cpp`](SkeletonXSkinXW_SSE.cpp.md) · [`xrCore/Animation/Bone.hpp`](../../xrCore/Animation/Bone.hpp.md)
**Used by** — [`SkeletonX.cpp`](SkeletonX.cpp.md) · [`SkeletonXSkinXW_CPP.cpp`](SkeletonXSkinXW_CPP.cpp.md) · [`SkeletonXSkinXW_SSE.cpp`](SkeletonXSkinXW_SSE.cpp.md)
**Tier floor** — T2: four declarations; the constraints live in the implementations.

## Purpose

Names the four skinning routines. Exactly one implementation of each is present in a build, chosen at compile time between the portable and the 4-wide file; the seam is deliberately a plain set of names with no dispatch object, because the choice is a build decision and never a runtime one.

## Exported units

```text
FUNCTION skin_1_influence(destination, source, vertex_count, bone_instances)
FUNCTION skin_2_influences(destination, source, vertex_count, bone_instances)
FUNCTION skin_3_influences(destination, source, vertex_count, bone_instances)
FUNCTION skin_4_influences(destination, source, vertex_count, bone_instances)
```

Each reads `vertex_count` vertices of its own source format and writes that many output vertices ([`SkeletonXVertRender.h`](SkeletonXVertRender.h.md)) into device-mapped memory, using the bone instances' *render* transforms. None allocates; none may be given overlapping ranges. The shared contract and the blend rule are in [`SkeletonXSkinXW_CPP.cpp`](SkeletonXSkinXW_CPP.cpp.md).
