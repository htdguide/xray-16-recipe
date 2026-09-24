# src/Layers/xrRender/KinematicsAddBoneTransform.hpp

> A standing per-bone offset the game layer can pin onto a skeleton, applied on top of whatever the animation produced.

**Needs** — [`utils/xrMiscMath/_quaternion.h`](../../xrCore/_quaternion.h.md)
**Used by** — [`Kinematics.h`](../../Include/xrRender/Kinematics.h.md) · [`KinematicsAnimated.h`](../../Include/xrRender/KinematicsAnimated.h.md)
**Tier floor** — T3: a record and four setters, all pure math.

## Purpose

Lets the game layer permanently bend one bone of a model without touching the animation data — the canonical use is aiming a weapon's bone chain or fixing a mis-authored attachment. The offset is authored as Euler angles plus a translation, because that is what the configuration files and the script layer express, and stored as a finished transform so the skeleton solve does no angle conversion per frame.

This is a late addition to the engine rather than an original design element: it sits beside the skeleton rather than inside it, and the skeleton applies a list of these after its own solve.

## State

```text
RECORD AdditionalBoneTransform
  bone_id       : int (16-bit)    # which bone; the all-ones value means "unset"
  global_axes   : bool            # see below
  transform     : matrix 4x4      # rotation and translation in one
```

**Invariants**

- `transform` is always a valid rigid transform: identity at construction, and every setter writes a rotation or a translation into it without ever composing the two in the wrong order. Rotation setters overwrite the rotation part; the translation setter overwrites the translation part. They are therefore order-independent, which is what the callers assume.
- `global_axes` records *which rotation convention the author used*, not a runtime mode. It is true when the angles were applied about the model's fixed axes in x, y, z order, and false when they were applied as yaw-pitch-roll about the bone's own axes. The skeleton reads it to decide whether to compose this offset before or after the bone's animated rotation. Storing the flag alongside the already-built matrix is redundant-looking and is not: the matrix cannot tell you which composition order it was built for.

## `set_rotation_global(x, y, z)`

**Contract** — builds the rotation from three angles applied about the model's fixed axes, in x-then-y-then-z order, and marks the offset as global. Replaces any previous rotation.

## `set_rotation_local(x, y, z)`

**Contract** — builds the rotation from the three angles read as yaw, pitch and roll about the bone's own axes — note the argument order is x, y, z but they are consumed as *pitch, yaw, roll*, so the caller passes pitch first. Marks the offset as local.

**Notes** — The swapped argument order is a real trap and is not documented anywhere in the original. It exists because the configuration files that feed this spell the offset as a vector and the engine's yaw-pitch-roll constructor takes yaw first. A rebuild should name the parameters rather than preserve the positional order.

## `set_position_offset(x, y, z)`

**Contract** — writes the translation part, leaving the rotation alone.
