# src/Layers/xrRender/SkeletonX.h

> Declares the skinned sub-mesh and the deformation modes it can be in, implemented in [`SkeletonX.cpp`](SkeletonX.cpp.md).

**Needs** — [`SkeletonX.cpp`](SkeletonX.cpp.md) · [`SkeletonCustom.h`](SkeletonCustom.h.md) · [`SkeletonXVertRender.h`](SkeletonXVertRender.h.md) · [`xrCDB/Intersect.hpp`](../../xrCDB/Intersect.hpp.md)
**Used by** — [`FSkinned.cpp`](FSkinned.cpp.md) · [`FSkinned.h`](FSkinned.h.md) · [`ModelPool.cpp`](ModelPool.cpp.md) · [`SkeletonAnimated.cpp`](SkeletonAnimated.cpp.md) · [`SkeletonCustom.cpp`](SkeletonCustom.cpp.md) · [`SkeletonX.cpp`](SkeletonX.cpp.md)
**Tier floor** — T1: it declares the overlapped mode-specific storage and the shared-array handles whose identity is the sharing key.

## Purpose

Declares the skinned sub-mesh: its state, its eleven render modes, and the abstract operations a concrete sub-mesh type must fill in. The deformation decision, the frozen vertex formats, the bone-matrix upload convention and every algorithm are in [`SkeletonX.cpp`](SkeletonX.cpp.md).

The header's own contribution is the **abstract/concrete split**: this type carries everything that does not depend on whether the sub-mesh is a plain mesh or a progressive one, and leaves four operations for the concrete type to supply — device-side load, per-bone face collection, decal fill and bone picking. The two concrete types differ only in that a progressive mesh's index buffer changes with the level of detail, so its face lists and its picking must consult the current detail level.

## Exported units

- `CSkeletonX` — the skinned sub-mesh: shared vertex arrays by influence count, the used-bone list, the render mode and its overlapped mode-specific fields.
- The render mode set — software, single-bone, and 1- to 4-influence hardware skinning, each hardware mode in a plain and a high-quality form.
- `_Load` — read the vertex chunk and choose the mode.
- `_Render` · `_Render_soft` — issue the draw, uploading bone matrices or skinning into the dynamic stream.
- `_Copy` · `SetParent` · `AfterLoad` — clone, and attach to the owning skeleton in a second step.
- `has_visible_bones` — whether any bone this mesh hangs from is currently visible.
- `_PickBoneSoft1W` … `4W` and the abstract `PickBone` — ray against this mesh's deformed faces.
- `_FillVerticesSoft1W` … `4W` and the abstract `FillVertices` — project a decal onto the deformed mesh.
- `EnumBoneVertices` — abstract: visit the vertices a given bone influences.
- `pick_bone` — the influence-count-independent body of the picking search, parameterized by vertex format.
- `get_pos_bones` — the deformed position of one vertex, one form per influence count.
- `vertRenderFVF` — the packed format identifier for the software skinning output: position, normal, one texture coordinate pair.
