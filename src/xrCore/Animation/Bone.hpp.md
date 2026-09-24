# src/xrCore/Animation/Bone.hpp

> The skeleton's data model: what a bone is, what its joint allows, what shape it collides with, and how a skinned vertex names its bones.

**Needs** — [`Bone.cpp`](Bone.cpp.md) · [`BoneEditor.cpp`](BoneEditor.cpp.md) · [`../_obb.h`](../_obb.h.md) · [`../_sphere.h`](../_sphere.h.md) · [`../_cylinder.h`](../_cylinder.h.md) · [`../_flags.h`](../_flags.h.md) · [`../FixedVector.h`](../FixedVector.h.md) · [`../xrstring.h`](../xrstring.h.md) · [Seam: Rigid-body dynamics](../../../SYSTEM-REQUIREMENTS.md#seam-rigid-body-dynamics)
**Used by** — [`Kinematics.h`](../../Include/xrRender/Kinematics.h.md) · [`SkeletonCustom.cpp`](../../Layers/xrRender/SkeletonCustom.cpp.md) · [`SkeletonCustom.h`](../../Layers/xrRender/SkeletonCustom.h.md) · [`SkeletonRigid.cpp`](../../Layers/xrRender/SkeletonRigid.cpp.md) · [`SkeletonX.cpp`](../../Layers/xrRender/SkeletonX.cpp.md) · [`SkeletonXSkinXW.h`](../../Layers/xrRender/SkeletonXSkinXW.h.md) · [`SkeletonXSkinXW_CPP.cpp`](../../Layers/xrRender/SkeletonXSkinXW_CPP.cpp.md) · [`SkeletonXSkinXW_SSE.cpp`](../../Layers/xrRender/SkeletonXSkinXW_SSE.cpp.md) · [`Bone.cpp`](Bone.cpp.md) · [`BoneEditor.cpp`](BoneEditor.cpp.md) · [`Motion.cpp`](Motion.cpp.md) · [`Motion.hpp`](Motion.hpp.md) · [`SkeletonMotions.hpp`](SkeletonMotions.hpp.md) · [`xr_collide_form.cpp`](../../xrEngine/xr_collide_form.cpp.md) · _and 17 more_
**Tier floor** — T1: the skinned vertex records are frozen byte layouts handed straight to the graphics device as vertex buffers, with declared strides.

## Purpose

Defines the vocabulary three very different consumers share: the renderer, which skins meshes; the physics layer, which turns a skeleton into a ragdoll; and the tools, which author both. Because they share it, this header carries the substance — the joint model, the collision shape model and the vertex layouts are defined nowhere else. [`Bone.cpp`](Bone.cpp.md) carries only the serialization; [`BoneEditor.cpp`](BoneEditor.cpp.md) carries the authoring operations.

Two parallel bone types exist and the difference is the point: **shared bone data** is the immutable per-model description loaded once and used by every instance, and a **bone instance** is the per-object, per-frame result. A hundred creatures of one kind share one skeleton description and own a hundred instance arrays.

## State — skinned vertex layouts

**Frozen.** These are vertex-buffer contents. The field order is the byte order, packing is two-byte aligned, and the whole record is handed to the graphics device with the stride given below.

```text
RECORD VertexOneBone            # 60 bytes
  position  : real[3]
  normal    : real[3]
  tangent   : real[3]
  binormal  : real[3]
  u, v      : real
  bone      : int (32-bit)      # the single influencing bone

RECORD VertexTwoBones           # 64 bytes
  bone0     : int (16-bit)      # NOTE: the bone indices come FIRST here
  bone1     : int (16-bit)
  position  : real[3]
  normal    : real[3]
  tangent   : real[3]
  binormal  : real[3]
  weight    : real              # weight of bone0; bone1 gets 1 - weight
  u, v      : real

RECORD VertexThreeBones         # 70 bytes
  bone      : int (16-bit)[3]
  position, normal, tangent, binormal : real[3]
  weight    : real[2]           # third weight is 1 - w0 - w1
  u, v      : real

RECORD VertexFourBones          # 76 bytes
  bone      : int (16-bit)[4]
  position, normal, tangent, binormal : real[3]
  weight    : real[3]           # fourth weight is 1 - w0 - w1 - w2
  u, v      : real

# invariant: the LAST weight is never stored. It is one minus the sum of the
#   stored ones. A rebuild that stores all of them changes the stride.
# invariant: the one-bone form stores its index as 32 bits while every other
#   form uses 16. That asymmetry is frozen.
# invariant: tangent and binormal are stored, not derived, so the normal map
#   convention that shipped is reproduced exactly.
```

**Notes** — the two-bone form puts its indices before the position and the others put them after. There is no reason beyond the order the formats were added, and it is frozen.

The packing directive is two bytes, not the natural alignment, which is why the three-bone form is 70 rather than 72. Padding it up changes the stride and the shipped models decode as garbage.

## State — the joint model

The joint description is what the rigid-body layer turns into a constraint. It is stored with the model and is authored per bone.

```text
ENUM JointKind
  rigid    # no relative motion at all
  cloth    # unused in shipped data
  joint    # three independent angular limits — a ball joint with stops
  wheel    # one angular limit on the first axis, second axis free, third locked
  none     # free
  slider   # no rotation; linear travel along the third axis, limited

RECORD JointLimit
  range          : real[2]      # (low, high), in radians
  spring_factor  : real         # 1.0 is neutral
  damping_factor : real         # 1.0 is neutral

RECORD JointData
  kind            : JointKind
  limits          : JointLimit[3]   # per axis; for a wheel, [0] is the wheel
                                    # axis and [1] is the steering axis
  spring_factor   : real            # whole-joint, multiplied with the per-axis
  damping_factor  : real
  breakable       : bool
  break_force     : real            # newtons; zero means "no threshold set"
  break_torque    : real
  friction        : real
```

**Invariants** — the limits stored in the file use the **authoring sign convention**, and the physics layer uses the opposite one. The conversion is fixed and is applied at the interface rather than at load: the engine-facing low limit is the negation of the stored *high*, and the engine-facing high limit is the negation of the stored *low*. Getting this backwards silently inverts every joint stop in the game.

**Notes** — the three limits are declared as an array of three because the ball joint needs three; the wheel and slider kinds use only some of them, and *which* ones is not uniform (a slider's travel limit is in slot 1, not slot 2, even though it travels along the third axis). Those assignments are in [`BoneEditor.cpp`](BoneEditor.cpp.md) and are frozen by the shipped data.

The spring and damping values are expressed in the original rigid-body library's units ([Seam: Rigid-body dynamics](../../../SYSTEM-REQUIREMENTS.md#seam-rigid-body-dynamics)). Swapping libraries requires re-deriving them, which the seam warns about.

## State — the collision shape

```text
ENUM ShapeKind  none, box, sphere, cylinder

RECORD BoneShape
  kind     : ShapeKind          # stored as 16 bits
  flags    : int (16-bit)       # not_pickable | remove_after_break
                                #   | no_physics | no_fog_collider
  box      : oriented box       # 15 reals: rotation basis, half-extents, centre
  sphere   : centre + radius    # 4 reals
  cylinder : centre + axis + height + radius   # 8 reals

# The record carries ALL THREE shapes regardless of kind, because it is written
# and read as one raw block — see Bone.cpp. Changing that is a format change.
```

**Invariants** — a shape is *valid* only if its selected kind has non-degenerate dimensions: every half-extent non-zero for a box, a non-zero radius for a sphere, non-zero height and radius and a non-zero axis for a cylinder. A shape of kind `none` is trivially valid. An invalid shape produces a rigid body the solver cannot integrate.

**Notes** — the four flags are separate concerns crammed into one word: `not_pickable` excludes the shape from ray queries but not from physics; `no_physics` does the opposite; `remove_after_break` is a destruction rule; `no_fog_collider` excludes it from the volumetric-fog occlusion test. A rebuild should keep them independent.

## `IBoneData` — the interface the consumers see

**Contract** — this is the read-only view the renderer and the physics layer are given, and it is the contract a rebuild must satisfy. An implementor must answer: its own index and its parent's; its children by index and how many there are; its joint data; its **bind transform** (the bone's rest pose relative to its parent); its collision shape and its oriented bounding box; its centre of mass and mass; its surface-material name and index; and its low and high angular limits per axis, *already converted to the engine's sign convention*.

**Invariants** — `bind_transform` is relative to the parent, not to the model. The model-relative transform is composed by walking the hierarchy, and that walk is what `CalculateM2B` does.

**Notes** — two classes implement this interface: the authoring bone and the shared runtime bone. They answer the same questions with different storage, and one of them — the authoring bone — deliberately returns "no material index" and makes the caller resolve the name through the material table, because the authoring tool has no material table loaded.

## `CBoneInstance` — the per-object result

```text
RECORD BoneInstance                       # 16-byte aligned; one per bone per object
  transform        : matrix               # final, model space
  render_transform : matrix               # bind-inverse then final — what the
                                          # vertex stage actually multiplies by
  callback         : optional<function>   # an override hook
  callback_param   : reference
  callback_kind    : int                  # dummy | physics | custom
  callback_wins    : bool                 # if set, skip animation for this bone
  param            : real[4]              # free per-bone channel for the hook
```

**Contract** — the animation pass fills `transform` from the blended motions unless a bone carries an override that declares it wins, in which case the animation for that bone is not even evaluated and the hook writes the transform itself. That is how a ragdoll takes over a bone, and how inverse kinematics adjusts a foot.

**Invariants** — the record is explicitly aligned to sixteen bytes because the two matrices are multiplied with four-wide float instructions. There are several thousand of these per frame and the alignment is load-bearing for throughput, not correctness.

**Notes** — "callback wins" is named a *performance hint* in the source, and that is exactly right: correctness would be unaffected by evaluating the animation and then overwriting it. Skipping it is worth doing because a ragdoll overrides most of a skeleton.

The four free reals are a generic side channel from the game layer to the hook — used, among other things, to pass a blend factor to a procedural aiming adjustment.

## `CBone` — the authoring bone

**Contract** — the tools' representation: name, parent name, the weight-map name, a rest pose (offset, rotation as Euler angles in the *game's* axis order, and a length), a current pose, the derived transforms, and the authoring-side extras — joint data, collision shape, material name, mass and centre of mass. It implements the shared interface, and additionally supports the authoring operations in [`BoneEditor.cpp`](BoneEditor.cpp.md).

**Invariants** — the bone name and the parent name are lowercased on assignment. Skeleton hierarchies are matched by name across files — a motion bank names bones by string — so a case difference between the model and the animation bank would silently drop the bone's animation.

**Notes** — rotations are stored as Euler triples rather than quaternions, in a fixed axis order. That is a format fact, not a preference: the shipped data is Euler and re-interpreting the triple in a different order produces subtly wrong poses that look almost right.

## `CBoneData` — the shared runtime bone

**Contract** — the immutable per-model description: index, parent index, name, oriented bounding box, bind transform, the **model-to-bone** transform, collision shape, material name and index, joint data, mass, centre of mass, children, and a per-child list of face indices.

### `CalculateM2B`

**Contract** — compute every bone's model-to-bone transform in one pass over the hierarchy, given the root's parent transform.

```text
FUNCTION compute_model_to_bone(bone, parent_model_transform)
  bone.m2b := parent_model_transform COMPOSED WITH bone.bind_transform
  FOR EACH child IN bone.children
    compute_model_to_bone(child, bone.m2b)      # children see the un-inverted form
  bone.m2b := inverse(bone.m2b)                 # only then invert, in place
```

**Invariants** — the inversion happens **after** the children have been visited, because the children need the forward composition. Inverting first and passing the inverse down produces a skeleton that folds in on itself. This ordering is the single subtle thing in the skeleton setup.

**Notes** — the result is what skinning multiplies by: a vertex is authored in model space, so to move it with a bone you first bring it into that bone's space (the model-to-bone transform) and then out through the bone's current transform. Only the affine 3×4 part is composed; the fourth row is known.

The per-child face list is a mapping from bone to the triangles it influences, used by the tools to select geometry by bone and by the collision proxy builder. It is shared across instances and can be large.

## Constants

- **no bone** — the all-ones 16-bit value, used everywhere a bone index may be absent. Bone indices are 16 bits throughout, capping a skeleton at 65534 bones; the practical limit is far lower.
- **inverse-kinematics data version 1** — gates the joint-data record's layout; see [`Bone.cpp`](Bone.cpp.md).
- **bone parameter count 4** — the width of the free channel in a bone instance.
