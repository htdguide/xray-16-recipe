# src/xrCore/Animation/BoneEditor.cpp

> Authoring operations on a bone — moving its shape, rotating it within its joint limits — and the one function the *engine* needs from that set: clamping a pose to what the joint allows.

**Needs** — [`Bone.hpp`](Bone.hpp.md) · [`Envelope.hpp`](Envelope.hpp.md) · [`../_matrix.h`](../_matrix.h.md) · [`../_obb.h`](../_obb.h.md)
**Used by** — [`Bone.hpp`](Bone.hpp.md)
**Tier floor** — T2: transform composition and clamping; nothing device-facing.

## Purpose

Almost everything here belongs to the skeleton editor and can be dropped from a shipping rebuild. One thing cannot: the rule that turns a joint kind into a set of allowed degrees of freedom. That rule is the *meaning* of the joint-kind enumeration, it is written down nowhere else, and the physics layer depends on agreeing with it.

## `SJointIKData.clamp_by_limits`

**Contract** — given a desired rotation as an Euler triple in bone-local space, force it into the joint's allowed set, in place. What "allowed" means depends entirely on the joint kind, and this function *is* the definition.

```text
FUNCTION clamp_by_limits(joint, rotation)
  SELECT joint.kind
    CASE rigid:
      rotation := (0, 0, 0)                    # no motion at all
    CASE joint:                                # ball joint with three stops
      rotation.x := clamp(rotation.x, joint.limits[0].range)
      rotation.y := clamp(rotation.y, joint.limits[1].range)
      rotation.z := clamp(rotation.z, joint.limits[2].range)
    CASE wheel:
      rotation.x := clamp(rotation.x, joint.limits[0].range)
      rotation.y := 0                          # note: z is left UNTOUCHED
    CASE slider:
      rotation.x := 0
      rotation.y := 0
      rotation.z := clamp(rotation.z, joint.limits[1].range)   # note: slot 1
    OTHERWISE:                                 # none, cloth
      unchanged
```

**Invariants** — two of the slot assignments look wrong and are frozen:

- A **wheel** clamps its first axis and zeroes its second, but leaves its third alone. A wheel is meant to spin about one axis and steer about another; the third component is left free because a wheel's own rotation about the spin axis is unbounded and the caller supplies it separately.
- A **slider** clamps the third component against the limit stored in **slot 1**, not slot 2. The slider travels along the third axis but its travel limit is authored into the second limit slot. There is no reading of the source that makes this consistent; it is what the shipped data contains.

**Notes** — a substantial commented-out block enumerates six axis-pair wheel variants that were designed and never shipped. Their absence is the reason the wheel case looks incomplete.

## `CBone.ClampByLimits`

**Contract** — clamp the bone's current pose against its joint, doing the clamp in **bind-relative space** rather than in the bone's own space.

```text
FUNCTION clamp_pose(bone)
  bind        := rotation_from_euler(bone.rest_rotation)
  current     := rotation_from_euler(bone.pose_rotation)
  relative    := inverse(bind) COMPOSED WITH current
  euler       := euler_from_rotation(relative)
  clamp_by_limits(bone.joint, euler)
  relative    := rotation_from_euler(euler)
  bone.pose_rotation := euler_from_rotation(bind COMPOSED WITH relative)
```

**Invariants** — joint limits are **relative to the bind pose**, not absolute. A limit of zero to zero means "stay exactly at the rest pose", not "point along the model's axis". Clamping the absolute pose instead would snap every bone to the model origin's orientation.

The round trip through Euler angles and back is lossy near the axis-order singularity. That loss is baked into the shipped joint tuning and a rebuild that clamps quaternions directly will produce visibly different ragdolls.

## `CBone.BoneRotate`

**Contract** — rotate a bone by an angle about an axis, either in bind-pose space or in the bone's own space, clamping to the joint afterwards. In bind-pose space the axis components are added to the Euler triple directly and then clamped — a cheap approximation that is only valid for small increments. In local space the axis is first transformed by the bone's current pose, a proper rotation is composed, the result is converted back to Euler, and *then* the bind-relative clamp is applied.

**Notes** — the two branches are not the same operation at different bases; the first is an approximation the editor accepts because the user is dragging a slider. A rebuild need not reproduce it.

## `CBone.BoneMove`

**Contract** — translate a bone, which is meaningful only for a slider joint. The translation is brought into the bone's rest frame, restricted to the third axis, added, clamped against the slider's travel limit **offset by the rest position**, and brought back. Every other joint kind is silently ignored.

**Invariants** — the travel limit is relative to the rest offset, matching the rotational convention.

## `CBone.ShapeScale` / `ShapeRotate` / `ShapeMove`

**Contract** — adjust the bone's collision shape. Scaling adds to a box's half-extents, a sphere's radius, or a cylinder's height and radius, clamping each at a small epsilon so a shape never becomes degenerate — that clamp is what keeps [`SBoneShape.Valid`](Bone.cpp.md) true. Rotation applies an Euler rotation to a box's basis or a cylinder's axis and does nothing to a sphere. Movement translates the shape's centre. Each takes a flag selecting whether the amount is in the parent's frame, in which case it is first brought into the bone's frame.

**Notes** — scaling maps the amount's components onto different quantities per kind: a sphere uses only the first component, a cylinder uses the first for radius and the third for height. That is a user-interface decision, not a data fact.

## `CBone.Pick`

**Contract** — intersect a ray, given in the parent's frame, against the bone's collision shape, returning the hit distance. A bone with no shape is picked against a small fixed-radius sphere at its origin — 2.5 centimetres — so that a bone with no collision geometry is still selectable in the editor. That radius is a user-interface constant with no effect on the game.

## `CBone.BindRotate` / `BindMove`

**Contract** — adjust the bone's rest pose by adding to its rest rotation or rest offset. Unclamped: the rest pose is the frame the limits are measured against, so clamping it would be circular.
