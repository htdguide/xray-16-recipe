# src/xrGame/ik/IKLimb.h

> The surface of one leg: create it against a skeleton, update it once per frame, ask it
> how far down the body may be pushed.

**Needs** — [`limb.h`](limb.h.md) · [`IKFoot.h`](../IKFoot.h.md) · [`KinematicsAnimated.h`](../../Include/xrRender/KinematicsAnimated.h.md) · [`ik_anim_state.h`](../ik_anim_state.h.md) · [`ik_calculate_data.h`](../ik_calculate_data.h.md) · [`ik_limb_state.h`](../ik_limb_state.h.md) · [`ik_collide_data.h`](../ik_collide_data.h.md) · [`ik_limb_state_predict.h`](../ik_limb_state_predict.h.md)
**Used by** — [`IKLimbsController.cpp`](../IKLimbsController.cpp.md) · [`IKLimbsController.h`](../IKLimbsController.h.md) · [`IKLimb.cpp`](IKLimb.cpp.md) · [`ik_calculate_data.cpp`](../ik_calculate_data.cpp.md) · [`ik_calculate_data.h`](../ik_calculate_data.h.md) · [`ik_dbg_matrix.cpp`](../ik_dbg_matrix.cpp.md) · [`ik_limb_state.cpp`](../ik_limb_state.cpp.md) · [`ik_limb_state.h`](../ik_limb_state.h.md)
**Tier floor** — T2. A declaration of one object's surface.

## Purpose

Declares the surface implemented in [`IKLimb.cpp`](IKLimb.cpp.md), which carries all the
substance. The type is a value: it copies and assigns, because the per-object controller
one directory up holds limbs in a container it grows.

The public surface is small and the private surface is large — roughly twenty private
steps against eight public ones. That ratio is the file's one message: **a limb is driven
entirely by its owning controller through four calls per frame, and everything else is
the internal pipeline of one leg**, documented in the implementation twin.

## Exported units

Lifecycle:

- `Create` — bind to a skeleton: resolve the four chain bones, read the foot geometry,
  configure the solver from the bind pose and the authored joint limits.
- `Destroy` — releases nothing; the limb owns no external resource.

Per frame, called in this order by the controller:

- `Update` — read the animation: is the foot planted, when is the next footstep, where
  will the body be then.
- `SetGoal` — decide what the foot should reach for this frame, including the ground
  query and the rate-limited blend.
- `ObjShiftDown` — how far this limb permits the whole body to be lowered, given what it
  must still be able to reach. The controller takes the binding answer across all limbs.
- `SolveBones` / `ApplyState` — run the chain solve and write the resulting bone
  transforms.

Queries the controller and the prediction path use:

- `get_id`, `ref_bone`, `foot_step`, `time_to_footstep`, `footstep_shift` — identity, the
  currently chosen contact reference bone, and the predicted next step.
- `ref_bone_to_foot`, `transform` — the two frame conversions a caller outside the limb
  needs: contact-reference to foot, and bone-to-bone along the chain.
- `step_predict`, `foot_matrix_predict` — where this foot will be at a future time, used
  to start lowering the body before the foot lands.
