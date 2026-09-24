# src/Layers/xrRender/KinematicAnimatedDefs.h

> The four fixed capacities of the animation blender, and the reason each one is the number it is.

**Needs** — [`xrCore/Animation/SkeletonMotionDefs.hpp`](../../xrCore/Animation/SkeletonMotionDefs.hpp.md)
**Used by** — [`KinematicsAnimated.h`](../../Include/xrRender/KinematicsAnimated.h.md) · [`Animation.h`](Animation.h.md) · [`SkeletonAnimated.cpp`](SkeletonAnimated.cpp.md) · [`SkeletonAnimated.h`](SkeletonAnimated.h.md)
**Tier floor** — T3: four constants and one fixed-capacity list type. Nothing here needs manual memory; the capacities do need to be *fixed*, because the per-bone mixing code below them works in stack arrays sized by these numbers.

## Purpose

Fixes how much animation a single skinned model may have in flight at once. Every array in the blending path is sized from these constants, so they are not tuning knobs — raising one costs stack and pool space in the inner loop of every animated model in the frame.

## State

```text
MAX_BLENDED       = 16   # concurrently running animations that may affect one body part
MAX_CHANNELS      = 4    # independent mixing lanes (see Animation.h)
MAX_ANIM_SLOT     = 48   # distinct animation slots a model may reserve

MAX_BLENDED_POOL  = MAX_BLENDED * MAX_PARTS * MAX_CHANNELS   # = 16*4*4 = 256
```

`MAX_PARTS` (4) comes from the shared animation data model: a skeleton is divided into at most four body parts, each of which can be playing its own animation set.

**Invariants**

- `MAX_BLENDED_POOL` is the size of the per-model pool of running-animation records. It is the product of the three capacities because the worst case is every body part running a full blend stack on every channel at once. The pool is preallocated per model and an allocation that fails is a hard error, not a dropped animation — the caller has already been told the animation started.
- The per-bone gather array is sized `MAX_BLENDED * MAX_CHANNELS` (64), one entry per running animation that could touch one bone. This is the list type the skeleton's bone solve fills and then sorts.

**Notes**

Sixteen is a *generous* ceiling chosen so that authored content never hits it, not a measured limit; the shipped game rarely exceeds four simultaneous blends on one part. Four channels is not generous — it is exactly the number of mixing rules the system defines, and adding a fifth means defining its rule (see [`Animation.h`](Animation.h.md)). A rebuild may make the blend stack a growable list, but should keep the ceiling as an assertion: an unbounded blend stack on one bone is always a leak in the caller, never a legitimate state.
