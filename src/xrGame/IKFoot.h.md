# src/xrGame/IKFoot.h

> Declares one foot's derived geometry and its ground-contact correction, implemented in [`IKFoot.cpp`](IKFoot.cpp.md).

**Needs** — [`ik_calculate_data.h`](ik_calculate_data.h.md) · [`ik_foot_collider.h`](ik_foot_collider.h.md) · [`IKFoot_inl.h`](IKFoot_inl.h.md)
**Used by** — [`IKFoot.cpp`](IKFoot.cpp.md) · [`IKFoot_inl.h`](IKFoot_inl.h.md) · [`IKLimb.cpp`](ik/IKLimb.cpp.md) · [`IKLimb.h`](ik/IKLimb.h.md)
**Tier floor** — T3: a declaration only

## Purpose

Declares `CIKFoot` and the small record that pairs a vector with the bone whose space it
is expressed in — the distinction the whole file turns on, because a foot is two bones and
a vector attached to one is not the same vector in the other. Substance is in
[`IKFoot.cpp`](IKFoot.cpp.md); the small conversions are inline in
[`IKFoot_inl.h`](IKFoot_inl.h.md).

Exported units:

- `local_vector` — a vector plus the bone index (ankle or toe) it belongs to.
- `CIKFoot` — one foot.
  - `Create` — bind to a skeleton and a configuration section, then derive the toe and
    heel from the mesh.
  - `set_ref_bone`, `get_ref_bone`, `ref_bone` — which of the two foot bones the
    placement is expressed at, detected from the live pose or set explicitly.
  - `ToePosition`, `HeelPosition`, `FootNormal` — the derived contact points and the
    sole's facing, in the reference bone's space.
  - `SetFootGeom` — build the three world points the ground query is cast against.
  - `Collide` — run the ground query and fill a contact record.
  - `GetFootStepMatrix` (two forms) — the correction: the transform that lays the foot on
    the surface, plus a classification of how it had to move.
  - `ref_bone_to_foot`, `foot_to_ref_bone` — frame conversions between the ankle and the
    reference bone, identity when they are the same.
  - `Kinematics` — the skeleton this foot belongs to.
- Private: `CollideFoot` (how far the foot may rotate before passing through the ground),
  `rotate` (lay the sole flat, clamped), `make_shift` (plant it along the cast
  direction), `set_toe` (the mesh-walking derivation), and the two bind-pose transform
  helpers.
