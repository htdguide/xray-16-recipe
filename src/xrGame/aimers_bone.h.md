# src/xrGame/aimers_bone.h

> Declares the chain aimer: aiming a target by spreading one correction across a fixed-length chain of bones.

**Needs** — [`aimers_base.h`](aimers_base.h.md) · [`aimers_bone_inline.h`](aimers_bone_inline.h.md) · [`Include/xrRender/Kinematics.h`](../Include/xrRender/Kinematics.h.md) · [`game_object_space.h`](game_object_space.h.md)
**Used by** — [`aimers_bone_inline.h`](aimers_bone_inline.h.md) · [`sight_manager.cpp`](sight_manager.cpp.md)
**Tier floor** — T3: a declaration.

## Purpose

Declares the surface implemented in [`aimers_bone_inline.h`](aimers_bone_inline.h.md), which
holds the substance. The chain length is fixed at the call site — a caller asks for a
two-bone or three-bone aimer — so the whole correction set is a fixed-size array with no
allocation, which matters because an aimer is built and thrown away inside a per-frame
decision.

Exported units:

- **construction** from an object, a motion, a first-or-last-frame flag, a target, and the
  chain's bone names in order from the base of the chain to its tip;
- **`get_bone`** — the correction computed for one link of the chain, indexed from the base.

The aimer computes its whole answer during construction; there is no separate solve step.
