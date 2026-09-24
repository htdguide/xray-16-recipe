# src/Layers/xrRender/SkeletonCustom.h

> Declares the rigid skeleton visual and the skinned-decal record, implemented in [`SkeletonCustom.cpp`](SkeletonCustom.cpp.md) and [`SkeletonRigid.cpp`](SkeletonRigid.cpp.md).

**Needs** — [`FHierrarhyVisual.h`](FHierrarhyVisual.h.md) · [`Include/xrRender/Kinematics.h`](../../Include/xrRender/Kinematics.h.md) · [`xrCore/Animation/Bone.hpp`](../../xrCore/Animation/Bone.hpp.md) · [`Shader.h`](Shader.h.md)
**Used by** — [`FSkinned.cpp`](FSkinned.cpp.md) · [`ModelPool.cpp`](ModelPool.cpp.md) · [`SkeletonAnimated.cpp`](SkeletonAnimated.cpp.md) · [`SkeletonAnimated.h`](SkeletonAnimated.h.md) · [`SkeletonCustom.cpp`](SkeletonCustom.cpp.md) · [`SkeletonRigid.cpp`](SkeletonRigid.cpp.md) · [`SkeletonX.cpp`](SkeletonX.cpp.md) · [`SkeletonX.h`](SkeletonX.h.md) · [`WallmarksEngine.cpp`](WallmarksEngine.cpp.md) · [`WallmarksEngine.h`](WallmarksEngine.h.md) · [`r__dsgraph_build.cpp`](r__dsgraph_build.cpp.md) · [`r__dsgraph_render.cpp`](r__dsgraph_render.cpp.md)
**Tier floor** — T1: it declares the per-instance bone arrays as raw parallel allocations and the decal record with an explicit size budget.

## Purpose

Declares the concrete model type that implements the skeleton interface from [`Include/xrRender/Kinematics.h`](../../Include/xrRender/Kinematics.h.md): a hierarchy visual whose children are skinned sub-meshes, plus a bone hierarchy shared with every clone and a bone-instance array private to each. The state, its sharing rules and every algorithm are in [`SkeletonCustom.cpp`](SkeletonCustom.cpp.md); the pose solve is in [`SkeletonRigid.cpp`](SkeletonRigid.cpp.md), which declares no type of its own.

One decision belongs to the header alone: the decal record is laid out to fit a **64-byte budget**, annotated field by field. Decals are created in bursts on every impact and walked every frame, so the record was sized to a cache line on purpose.

## Exported units

- `CKinematics` — the rigid skeleton visual: the concrete implementor of the skeleton interface.
  - Loading and lifecycle: `Load`, `Copy`, `Spawn`, `Depart`, `Release`, `LL_Validate`.
  - Bone access: `LL_BoneID` (two string forms), `LL_BoneName_dbg`, `LL_GetBoneInstance`, `LL_GetData`, `GetBoneData`, `LL_GetBoneData`, `LL_BoneCount`, `LL_VisibleBoneCount`, `LL_GetTransform`, `LL_GetTransform_R`, `LL_GetBox`, `GetBox`, `LL_GetBindTransform`, `LL_GetBoneGroups`, `LL_UserData`, `LL_Bones`.
  - Root and visibility: `LL_GetBoneRoot`, `LL_SetBoneRoot`, `LL_GetBoneVisible`, `LL_SetBoneVisible`, `LL_GetBonesVisible`, `LL_SetBonesVisible`, `Visibility_Update`.
  - The solve: `CalculateBones`, `CalculateBones_Invalidate`, `Bone_Calculate`, `CLBone`, `BuildBoneMatrix`, `OnCalculateBones`, `BoneChain_Calculate`, `Bone_GetAnimPos`.
  - Script-authored offsets: `LL_AddTransformToBone`, `LL_ClearAdditionalTransform`, `CalculateBonesAdditionalTransforms`.
  - Queries: `PickBone`, `EnumBoneVertices`.
  - Decals: `AddWallmark`, `CalculateWallmarks`, `RenderWallmark`, `ClearWallmarks`.
  - Notification: `Callback`, `SetUpdateCallback`, `SetUpdateCallbackParam`, `GetUpdateCallback`, `GetUpdateCallbackParam`.
  - Downcasts: `dcast_PKinematics`, `dcast_PKinematicsAnimated` (always absent here), `dcast_RenderVisual`.
  - Diagnostics: `mem_usage` (shared data counted only for a base model), `DebugRender`, `getDebugName`.
- `CSkeletonWallmark` — one decal stuck to skin: material, contact point, birth time, model- and world-space bounds, and its triangles recorded in bind pose with their skinning influences. `Similar` decides whether a new decal replaces it.
- `PCKinematics` — the null-tolerant downcast from a visual to a skeleton.
- `UCalc_Mutex` — the process-wide lock every pose solve serializes on.
