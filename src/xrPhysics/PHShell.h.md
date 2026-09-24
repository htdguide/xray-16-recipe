# src/xrPhysics/PHShell.h

> Declares the physical identity of one game object: the set of bodies, the joints between them, the collision space they share, and the transform that maps the object's origin onto the body the engine treats as its root.

**Needs** — [`PHShell.cpp`](PHShell.cpp.md) · [`PHShellActivate.cpp`](PHShellActivate.cpp.md) · [`PHShellNetState.cpp`](PHShellNetState.cpp.md) · [`PHElement.h`](PHElement.h.md) · [`PHJoint.h`](PHJoint.h.md) · [`PHShellSplitter.h`](PHShellSplitter.h.md) · [`PHMoveStorage.h`](PHMoveStorage.h.md) · [`PHObject.h`](PHObject.h.md) · [`PhysicsShell.h`](PhysicsShell.h.md) · [`PHDefs.h`](PHDefs.h.md) · [Seam: Rigid-body dynamics](../../SYSTEM-REQUIREMENTS.md#seam-rigid-body-dynamics)
**Used by** — [`ActorCameraCollision.cpp`](ActorCameraCollision.cpp.md) · [`PHElement.cpp`](PHElement.cpp.md) · [`PHElementInline.h`](PHElementInline.h.md) · [`PHElementNetState.cpp`](PHElementNetState.cpp.md) · [`PHFracture.cpp`](PHFracture.cpp.md) · [`PHJoint.cpp`](PHJoint.cpp.md) · [`PHJoint.h`](PHJoint.h.md) · [`PHShell.cpp`](PHShell.cpp.md) · [`PHShellActivate.cpp`](PHShellActivate.cpp.md) · [`PHShellNetState.cpp`](PHShellNetState.cpp.md) · [`PHShellSplitter.cpp`](PHShellSplitter.cpp.md) · [`PHShellSplitter.h`](PHShellSplitter.h.md) · [`PHSplitedShell.cpp`](PHSplitedShell.cpp.md) · [`PHSplitedShell.h`](PHSplitedShell.h.md) · _and 4 more_
**Tier floor** — T1: it owns a collision space and a list of solver bodies, and is the type the engine hands across the dynamics-library boundary.

## Purpose

Declares the surface implemented in [`PHShell.cpp`](PHShell.cpp.md) (state, queries, building from a
skeleton), [`PHShellActivate.cpp`](PHShellActivate.cpp.md) (the lifecycle) and
[`PHShellNetState.cpp`](PHShellNetState.cpp.md) (the wire form).

**A shell is what the game means by "this object has physics."** Elements are bodies and joints are
constraints, but neither is addressable from the game layer: the game says "open this door", "shoot
this crate", "this corpse becomes a ragdoll", and every one of those is addressed to a shell. The
split between shell and element exists because the game's vocabulary is object-shaped and the
solver's is body-shaped, and something must translate.

The type carries four roles fused together, and a rebuild should keep them apart: the abstract
physics-object interface the engine holds; a simulated object in the world's step and collision
lists; the owner of a collision space; and the owner of a skeleton's bone callbacks.

## State

```text
RECORD Shell
  elements         : list<Element>       # in skeleton-hierarchy order; [0] is the ROOT
  joints           : list<Joint>         # in the same order
  space            : collision space     # this shell's shapes, grouped so they never collide
                                         # with each other
  splitter_holder  : optional<SplitterHolder>   # present only while breakable
  traced_geoms     : MoveStorage         # shapes whose motion is swept, not just sampled
  object_in_root   : transform           # the game object's origin, in the ROOT element's frame
  animator         : optional<ShellAnimator>    # present only for data-driven animated shells
  active_count     : int
  flags            : set of {active, activating, remove_character_collision_after_disable}
```

**Invariants** — the load-bearing ones:

- **Element and joint order is skeleton order**, parents before children. Every range-based operation
  in the breaking machinery depends on it; see [`PHFracture.h`](PHFracture.h.md).
- **Element zero is the root.** The shell's transform, its velocity readout, its impulse target and
  its network identity all come from it.
- **`object_in_root` is the object-to-physics bridge.** The game object's origin is not the root
  body's origin — the root body sits at its own centre of mass, at the root bone, and the object's
  origin is wherever the model says. The transform between them is captured once at activation, from
  the animated pose, and applied on every read of the shell's placement.
- `activating` means the shell is active but no element has yet adopted its animated pose. `active`
  without `activating` is `isFullActive`, which is the state most operations require.

## the ownership contract

**Contract** — this is the chapter's central question and the shell is where it is answered.

```text
# who owns a bone's transform
#   no physics callback              → animation owns it
#   callback installed, element set  → this shell's element owns it and writes it each frame
#   no callback, element set         → the bone rides that element; animation still poses it,
#                                      physics supplies the frame it is posed in
```

**Exported units** — `SetCallbacks` installs both kinds across the whole skeleton; `ZeroCallbacks`
removes every physics callback; `ResetCallbacks` re-installs them for a bone subtree under a
visibility mask; `EnabledCallbacks` switches the whole thing on or off; `SetBonesCallbacksOverwrite`
controls whether a physics-written bone is composed with its parent again.

`GetBonesCallback` and `GetStaticObjectBonesCallback` hand out the two callback entry points. The
static-root variant asserts if asked for — it is dead; see [`PHShell.cpp`](PHShell.cpp.md).

## Exported units

**Building** — `build_FromKinematics` (from a skeleton, keeping the skeleton),
`preBuild_FromKinematics` (the same, then forgetting it), `add_Element`, `add_Joint`, `CreateSpace`,
`PlaceBindToElForms`, `ActivatingBonePoses`.

**Lifecycle** — the four `Activate` overloads, `Build`, `RunSimulation`, `PresetActive`,
`PureActivate`, `AfterSetActive`, `Deactivate`. Contracts in
[`PHShellActivate.cpp`](PHShellActivate.cpp.md).

**Step participation** — `PureStep`, `CollideAll`, `PhTune`, `PhDataUpdate`, `Update`,
`UpdateRoot`, `AnimatorOnFrame`, `StepFrameUpdate` (empty), `InitContact` (empty — a shell adds
nothing to a contact; the element does).

**Sleep and freezing** — `Enable`, `Disable`, `EnableObject`, `DisableObject`, `isEnabled`,
`Freeze`, `UnFreeze`, `FreezeContent`, `UnFreezeContent`, `set_DisableParams`,
`vis_update_activate`, `vis_update_deactivate`.

**Collision filtering** — `DisableCollision`, `EnableCollision`, `DisableCharacterCollision`,
`SetRemoveCharacterCollLADisable`, `SetIgnoreStatic`, `SetIgnoreDynamic`, `SetRagDoll`,
`SetIgnoreRagDoll`, `SetIgnoreAnimated`, `SetSmall`, `SetIgnoreSmall`, `RegisterToCLGroup`,
`GetCLGroup`, `IsGroupObject`, `collide_bits`, `collide_class_bits`. The policy is in
[`PHCollideValidator.h`](PHCollideValidator.h.md).

**Mass** — `setMass` (by volume), `setMass1` (evenly), `setDensity`, `getMass`, `getDensity`,
`getVolume`, `MassAddBox`, `setEquelInertiaForEls`, `addEquelInertiaToEls`, `SmoothElementsInertia`,
`get_Extensions`.

**Dynamics** — `applyForce`, `applyImpulse`, `applyImpulseTrace` (with and without a bone),
`applyHit`, `applyGravityAccel`, `setForce`, `setTorque`, `set_JointResistance`,
`set_DynamicLimits`, `set_DynamicScales`, `set_LinearVel`, `set_AngularVel`, `get_LinearVel`,
`get_AngularVel`, `CutVelocity`, `SetAirResistance`, `GetAirResistance`, `set_ApplyByGravity`,
`get_ApplyByGravity`.

**Placement** — `SetTransform`, `TransformPosition`, `SetGlTransformDynamic`,
`GetGlobalTransformDynamic`, `GetGlobalPositionDynamic`, `InterpolateGlobalTransform`,
`InterpolateGlobalPosition`, `ObjectInRoot`, `ObjectToRootForm`, `SetObjVsShellTransform`,
`NetInterpolationModeON` / `OFF`.

**Lookup** — `get_Element` by bone identifier, by bone name or by store order;
`get_PhysicsParrentElement` (the nearest physics ancestor of a bone); `get_Joint` in the same three
ways; `get_GeomByID`; `NearestToPoint`; `get_ElementsNumber`; `get_JointsNumber`;
`get_ElementSync`; `Elements`.

**Breaking** — `isBreakable`, `isFractured`, `SplitProcess`, `SplitterHolder`,
`SplitterHolderActivate` / `Deactivate`, `BlockBreaking`, `UnblockBreaking`, `IsBreakingBlocked`,
`PassEndElements`, `PassEndJoints`, `DeleteElement`, `DeleteJoint`, `BoneIdToRootGeom`.

**Sweeping** — `AddTracedGeom`, `SetAllGeomTraced`, `ClearTracedGeoms`, `EnableGeomTrace`,
`DisableGeomTrace`, `HasTracedGeoms`, `MoveStorage`, `SetPrefereExactIntegration`.

**Material and callbacks** — `SetMaterial` by index or name, `set_ContactCallback`,
`set_ObjectContactCallback`, `add_` / `remove_ObjectContactCallback`, `set_CallbackData`,
`get_CallbackData`, `set_PhysicsRefObject`, `PhysicsRefObject`.

**Animation** — `ToAnimBonesPositions`, `AnimToVelocityState`, `SetAnimated`,
`CreateShellAnimator`, `PPhysicsShellAnimator`.

**Network** — `net_Import`, `net_Export`, `get_element_sync`, `get_elements_number`.

## `SetObjVsShellTransform`

**Contract** — records the *inverse* of a root bone's transform as the object-in-root bridge, and
clears the activating flag. Called once, by the root element, at its first bone evaluation after
activation.

```text
FUNCTION set_obj_vs_shell_transform(root_bone_transform)
  object_in_root := inverse(root_bone_transform)
  activating := false
```

**Notes** — the bone transform passed in is *object-relative* — the skeleton's own frame — so its
inverse maps root-bone space back to object space. Composing a body's world placement with it gives
the game object's world placement, which is the only thing the rest of the engine wants. This one
line is why a ragdoll's object origin stays attached to its root bone as the ragdoll flops, instead
of the object teleporting to the pelvis.
