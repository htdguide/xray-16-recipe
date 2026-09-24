# src/Layers/xrRender/SkeletonAnimated.h

> Declares the animated skeleton visual and the per-bone blend list, implemented in [`SkeletonAnimated.cpp`](SkeletonAnimated.cpp.md).

**Needs** — [`SkeletonCustom.h`](SkeletonCustom.h.md) · [`Animation.h`](Animation.h.md) · [`KinematicAnimatedDefs.h`](KinematicAnimatedDefs.h.md) · [`xrCore/Animation/SkeletonMotions.hpp`](../../xrCore/Animation/SkeletonMotions.hpp.md) · [`Include/xrRender/KinematicsAnimated.h`](../../Include/xrRender/KinematicsAnimated.h.md)
**Used by** — [`ModelPool.cpp`](ModelPool.cpp.md) · [`SkeletonAnimated.cpp`](SkeletonAnimated.cpp.md)
**Tier floor** — T1: it declares the blend pool and the per-bone blend lists as fixed-capacity inline arrays with an explicit packing, because a bone's whole blend list is walked in the pose inner loop.

## Purpose

Declares the model type that is both a skeleton and an animation player — the rigid skeleton from [`SkeletonCustom.h`](SkeletonCustom.h.md) plus motion banks, a blend pool and the channel mixer — satisfying the interface in [`Include/xrRender/KinematicsAnimated.h`](../../Include/xrRender/KinematicsAnimated.h.md). Its state, the fixed pool and partition sizes, and every algorithm are in [`SkeletonAnimated.cpp`](SkeletonAnimated.cpp.md).

## Exported units

- `CBlendInstance` — one bone's list of covering blends: `blend_add` (evicts the lowest-weighted blend when full, unless the newcomer is transient), `blend_remove`, `blend_vector`, `construct`.
- `CKinematicsAnimated` — the animated skeleton visual.
  - Lifecycle: `Load`, `Copy`, `Spawn`, `Release`, `IBoneInstances_Create` / `_Destroy`, `IBlend_Startup`, `IBlend_Create`.
  - Motion lookup: `LL_MotionID`, `ID_Cycle` and `ID_Cycle_Safe` (string and interned forms), `ID_FX`, `ID_FX_Safe`, `LL_PartID`, `LL_MotionsSlotCount`, `LL_MotionsSlot`, `LL_GetMotionDef`, `LL_GetRootMotion`, `LL_GetMotion`, `get_animation_length`, `partitions`.
  - Starting motions: `PlayCycle` (three forms), `LL_PlayCycle` (two forms), `PlayFX`, `PlayFX_Safe`, `LL_PlayFX`.
  - Stopping: `LL_FadeCycle`, `LL_CloseCycle`, `DestroyCycle`.
  - Time: `UpdateTracks`, `LL_UpdateTracks`, `LL_UpdateFxTracks`, `OnCalculateBones`.
  - Channels: `LL_SetChannelFactor`, `ChannelFactorsStartup`.
  - Pose: `BuildBoneMatrix` (overriding the rigid version), `LL_BuldBoneMatrixDequatize`, `LL_BoneMatrixBuild`.
  - Blend registration: `Bone_Motion_Start` / `Bone_Motion_Stop` (recursive, for effects) and their non-recursive counterparts (for cycles), `LL_GetBlendInstance`.
  - Inspection: `LL_PartBlendsCount`, `LL_PartBlend`, `LL_IterateBlends`, `blend_cycle`.
  - Driver hooks: `SetUpdateTracksCalback`, `GetUpdateTracksCalback`, `SetBlendDestroyCallback`, `GetBlendDestroyCallback`.
  - Script offsets forwarded to the rigid base: `LL_AddTransformToBone`, `LL_ClearAdditionalTransform`.
  - Downcasts: `dcast_PKinematicsAnimated` (present here), `dcast_PKinematics`, `dcast_RenderVisual`.
  - Diagnostics: `mem_usage`, `LL_MotionDefName_dbg`, `LL_DumpBlends_dbg`.
- `PKinematicsAnimated` — the null-tolerant downcast from a visual to an animation player.
