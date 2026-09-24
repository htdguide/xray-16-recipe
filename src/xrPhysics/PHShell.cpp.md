# src/xrPhysics/PHShell.cpp

> Turns a skeleton into a body-and-joint graph, and answers the whole game layer's questions about one physical object — including the two that matter: where is it, and who owns its bones.

**Needs** — [`PHShell.h`](PHShell.h.md) · [`PHElement.h`](PHElement.h.md) · [`PHElementInline.h`](PHElementInline.h.md) · [`PHJoint.h`](PHJoint.h.md) · [`PHShellBuildJoint.h`](PHShellBuildJoint.h.md) · [`PHShellSplitter.h`](PHShellSplitter.h.md) · [`PHFracture.h`](PHFracture.h.md) · [`PHJointDestroyInfo.h`](PHJointDestroyInfo.h.md) · [`PHCollideValidator.h`](PHCollideValidator.h.md) · [`PhysicsShellAnimator.h`](PhysicsShellAnimator.h.md) · [`PHObject.h`](PHObject.h.md) · [`Physics.h`](Physics.h.md) · [`SpaceUtils.h`](SpaceUtils.h.md) · [`MathUtils.h`](MathUtils.h.md) · [`ExtendedGeom.h`](ExtendedGeom.h.md) · [`IPhysicsShellHolder.h`](IPhysicsShellHolder.h.md) · [`Include/xrRender/Kinematics.h`](../Include/xrRender/Kinematics.h.md) · [Seam: Rigid-body dynamics](../../SYSTEM-REQUIREMENTS.md#seam-rigid-body-dynamics)
**Used by** — [`PHShell.h`](PHShell.h.md)
**Tier floor** — T1: it owns a collision space, builds mass tensors, and reads a skeleton's bind poses as laid out in the model file.

## Purpose

Three things live here and only the first two are interesting.

**Building a shell from a skeleton** — the one place where authored model data becomes a simulated
structure. Every decision about what becomes a body, what becomes a joint, what breaks, and what is
merely carried along is made in one recursive walk.

**The bone ownership machinery** — installing, clearing and resetting the callbacks that transfer a
bone's transform between animation and physics.

**Broadcast** — the long tail of "do this to every element", which is mechanical.

## building from a skeleton

**Contract** — `build_FromKinematics` walks the skeleton from its root and produces the element list,
the joint list, the fracture seams and the splitter list. Optionally fills a caller-supplied map
from bone identifier to the element and joint created for it. Requires a skeleton whose physics
data passes validation. Creates a splitter holder speculatively and deletes it if nothing turned
out to be breakable.

`preBuild_FromKinematics` is the same walk followed by forgetting the skeleton — used to price a
shell (its mass, its extents) without binding it to a model instance.

### the recursive walk — the decisions

```text
FUNCTION add_element_recursive(parent_element, bone_id, parent_frame, element_index)
  frame := parent_frame · bone.bind_transform         # absolute bind pose, accumulated down

  IF the bone is INVISIBLE THEN
    recurse into the children with THIS element unchanged
    RETURN                                            # invisible bones contribute nothing

  breakable := bone says breakable AND a parent element exists
               AND NOT (bone has no shape AND its joint is rigid)

  IF the bone has a shape, OR this is the root THEN
    IF the bone's joint is RIGID and a parent element exists THEN
      # --- the bone FUSES into its parent's body ---
      element := parent_element
      add this bone's shape and mass to it, placed relative to the parent's frame
      IF breakable THEN open a fracture seam here, recording the shape index and the
                        element and joint counts as its start
    ELSE
      # --- the bone becomes its OWN body ---
      element := a new element owning this bone
      element.frame := frame
      element.material := the bone's, or looked up by the bone's material name
      IF the bone has a shape THEN add it and set the mass about the bone's centre of mass
      add the element to the shell
      IF a parent element exists THEN
        joint := build_joint(bone, parent_element, element)      # see PHShellBuildJoint.h
        IF joint THEN
          record which of the parent's shapes the joint is anchored to
          add the joint ; IF breakable THEN make it breakable and add a joint splitter
  ELSE
    element := parent_element                         # a shapeless, non-root bone rides its parent

  IF the bone has a shape THEN tag the newly added shape with this bone's identity and flags
  record (bone_id → the shape) in the splitter holder's lookup

  recurse into every child with `element` as their parent

  IF breakable THEN close the seam: record the element, joint and shape counts as its END
  IF an element was created here AND it is breakable THEN add an element splitter
```

**Invariants** — everything in the breaking machinery rests on what this walk guarantees:

- **Elements, joints and shapes are all appended in depth-first skeleton order**, so any subtree is a
  contiguous range in all three lists. That is what makes a break a range operation; see
  [`PHFracture.h`](PHFracture.h.md).
- A seam is **opened before the recursion and closed after it**, so the range it records is exactly
  the subtree. The mass that accumulates while the seam is open lands on its second side, which is
  how the mass split is computed without a second pass.
- The splitter for an element is inserted at a *recorded position* rather than appended, so that
  splitters stay in the same order as the things they split.

**Notes** — the three decisions worth naming:

*A rigid joint means "not a joint at all".* A bone whose authored joint type is rigid does not get a
body; its shape is folded into its parent's body. That is how a model with fifty bones becomes an
object with six rigid parts. The authored joint type is therefore doing double duty: it says both
what kind of constraint to build and *whether to build a body*.

*Invisible bones are skipped entirely* but their children are not, and the frame accumulation
continues through them. Bone visibility is a per-instance mask, so the same model can build
different shells — this is how a damaged variant of a prop has fewer physical parts than an intact
one.

*Breakability is inherited from the joint data but requires a parent.* The root can never break off,
which is sound: there is nothing for it to break from.

The limit of 64 bones on a breakable model is asserted, and the reason is the visibility mask — it
is a 64-bit word, one bit per bone. A rebuild with a variable-length mask lifts the limit.

### `ClearBreakInfo`

**Contract** — strips every element's and joint's break criterion and destroys the splitter holder.
Run at the end of a build that produced no splitters, and available to the game to make an object
permanently unbreakable.

## the bone ownership machinery

**`SetCallbacks`** — the live path, and the shape of it is the contract:

```text
FUNCTION set_callbacks()
  FOR EACH element
    install the physics callback on the element's own bone
  FOR EACH bone in the skeleton
    IF it has no physics callback AND it is visible THEN
      ancestor := the nearest ancestor bone (or itself) carrying a physics callback
      IF ancestor exists THEN record that element on this bone WITHOUT a callback
```

**Invariants** — two distinct installations, and the difference is the whole ownership contract. A
bone with a *callback* is written by physics. A bone with only a *recorded element* is still posed
by animation, but anything asking "which body does this bone belong to" — a hit, a lookup, an
attachment — finds the right one. That second pass is why a shot at a character's forearm, which is
not a physics bone, still pushes the arm body.

**`ZeroCallbacks`** — walks the whole skeleton removing every physics callback, including the
reference-only ones.

**`ResetCallbacks`** — the subtree variant, under an explicit visibility mask, walking the skeleton
and advancing an element index in step with the same rule the build used (a shapeless or
rigid-jointed bone reuses the current element; anything else consumes the next one). Used when a
model's bone visibility changes and the shell must be re-bound without being rebuilt.

**Notes** — the index-walking form is fragile: it assumes the element list is in exactly the order
the same walk would produce, and asserts if it runs out. `SetCallbacksRecursive` is the same code
with a static index, is asserted-unreachable, and is dead — `SetCallbacks` replaced it with the
two-pass form above, which does not depend on the walk order at all. A rebuild should use the
two-pass form everywhere and delete the index walk, including in `ResetCallbacks`.

**`EnabledCallbacks`** — on, install everything and mark every element's own bone as overwriting its
parent; off, clear everything. The overwrite mark is what stops a physics-written absolute transform
from being composed with its parent again — without it, a ragdoll inflates.

## placement — the four ways to ask where a shell is

**Contract** — all four compose the *root element's* placement with `object_in_root`.

```text
FUNCTION interpolate_global_transform() -> transform
  refresh every element's cached frame from its interpolation blend
  RETURN root_element.frame · object_in_root

FUNCTION get_global_transform_dynamic() -> transform
  refresh every element's cached frame from the SOLVED state
  RETURN root_element.frame · object_in_root
```

**Invariants** — the interpolated form is for rendering, the dynamic form for reasoning. Mixing them
is how objects end up a fraction of a step out of step with their own collision.

**Notes** — both refresh *every* element, not just the root, because the caller is usually about to
read the elements too and the refresh is not free. The interpolated form additionally notices when
the shell's visibility-driven activity count has gone negative and tells the game object to stop
processing and re-index itself spatially — a piece of lifecycle bookkeeping riding in a getter,
which a rebuild should move out.

`InterpolateGlobalPosition` takes the shortcut of *adding* the object-in-root translation instead of
composing the transform, which is only correct when the root's rotation is identity. For a settled
object it is; for a tumbling one the reported position wobbles. It is used for cheap proximity
queries where that does not matter.

**`ObjectToRootForm`** — the inverse question: given where the *object* should be, where must the
root body go? Composes the object-in-root bridge with the root's mass-centre shift and inverts.

**`SetGlTransformDynamic`** — move a whole shell to a given placement by computing the delta from
where it is now and applying that delta to every element, clearing motion history so the move is not
swept as a real motion.

## mass

**`setMass`** — distribute a total across elements **by volume**, so a big part is heavy.
**`setMass1`** — divide it **equally**, regardless of size. Both exist and both are used; the second
is for objects whose parts should feel alike regardless of shape.

**`SmoothElementsInertia`** — blends every element's mass tensor toward the shell's average by a
factor, preserving each element's own centre of mass.

```text
FUNCTION smooth_elements_inertia(k)
  average := (sum of every element's mass tensor) · k / element_count
  FOR EACH element
    keep its centre of mass aside
    scale its tensor by (1 - k)
    add the average
    restore its centre of mass
```

**Notes** — this is a stability tool, not physics. A solver's convergence degrades sharply when
adjacent constrained bodies have very different inertias — a ragdoll with a heavy torso and
featherweight fingers jitters at the wrists. Pulling every element's inertia toward the mean trades
a small loss of realism for a large gain in solver behaviour, and it is the standard fix for the
problem. The centre of mass is deliberately *not* blended: moving it would move the bodies.

## the step

**`PhTune`** — pre-solve, broadcast to elements.

**`PhDataUpdate`** — post-solve, and it makes two shell-level decisions:

```text
FUNCTION ph_data_update(step)
  run every element's post-solve pass
  IF EVERY element's body is asleep THEN
    disable the whole shell and put it in the recently-deactivated list
  IF the root body has left the world's boundaries THEN disable the shell entirely
```

**Invariants** — sleep is **all or nothing across a shell**. One awake body keeps the whole object
awake, because a partly-asleep constrained assembly gives the solver an inconsistent system and the
object visibly tears.

**Notes** — the out-of-boundaries check is a containment measure: an object that has escaped the
world through a solver failure is switched off rather than allowed to keep producing non-finite
numbers that spread to everything it touches. Only the *root* is checked, which is enough because
the joints keep the rest nearby, and is cheap.

**`Update`** — the per-frame (not per-step) refresh: clears the activating flag, updates every
element, and adopts the root's frame as the shell's.

**`UpdateRoot`** — the cheap variant that only refreshes the shell's transform from the root's
interpolation, and only when the shell is fully active.

## impulses and hits

**Contract** — three shapes, distinguished by what they know about where the blow landed.

```text
apply_force(direction, magnitude)       # distributed across elements BY MASS, so the object
                                        # accelerates as a whole rather than folding
apply_impulse(direction, magnitude)     # to the ROOT element only
apply_impulse_trace(point, direction, magnitude)            # to the root, at a point
apply_impulse_trace(point, direction, magnitude, bone_id)   # to the element that OWNS that bone
```

**Invariants** — every one of them refuses on an inactive shell and wakes the shell afterwards.

**Notes** — the bone-directed form resolves the bone through its *callback*: it reads the element
recorded on that bone and refuses if the bone is not physics-owned at all. That is the second
installation from `SetCallbacks` paying off — a hit on a non-physics bone still finds the body the
bone rides. See [`PHElement.cpp`](PHElement.cpp.md) for what the element does with it.

`applyGravityAccel` multiplies the acceleration by the element count before broadcasting, because
each element then scales by its own share. The arithmetic is correct only when the elements have
equal mass; for an unequal shell the total is wrong. It is used for buoyancy-like effects where the
error is invisible.

## lookup

**`get_Element(bone_id)`** — two paths, and the fast one is the interesting one: if the shell is
active and the bone carries a physics callback, the element is read straight off the bone. Otherwise
the element list is scanned for one owning that bone. The callback *is* the index.

**`get_PhysicsParrentElement`** — walk up the bone hierarchy until an element is found. The answer to
"which body does this bone belong to" for bones that are not themselves physics bones.

**`NearestToPoint`** — linear scan over elements by distance, with an optional filter. Used to decide
which limb a grab or a hit attaches to.

**`BoneIdToRootGeom`** — bone identifier to the shape index that seam's break is anchored at, through
the splitter holder's lookup.

## collision grouping

**Contract** — every shape of a shell lives in one collision space, and a space's members never
collide with each other. Filtering *between* shells is a separate mechanism — classes and groups,
in [`PHCollideValidator.h`](PHCollideValidator.h.md) — and the shell exposes the setters:
ignore the static world, ignore dynamics, ignore other ragdolls, ignore animated shells, ignore small
objects, join a named group.

**Notes** — the space is what makes a ragdoll possible at all: a character's thigh and shin overlap
at the knee by construction, and without the grouping they would push each other apart forever. The
cost is that a shell can never self-collide, so a ragdoll's arm passes through its own chest. Every
engine of this design makes the same trade.

`SetRemoveCharacterCollLADisable` arms a one-shot: when the shell next falls asleep, stop colliding
with characters. This is how a settled corpse stops blocking the player's path without being
removed.

## sweeping

**Contract** — `AddTracedGeom` marks one shape as *swept*: its motion between steps is traced rather
than sampled at the endpoints. `SetAllGeomTraced` marks every shape. `ClearTracedGeoms` removes them
all.

**Notes** — this is the tunnelling defence, and it is opt-in per shape because sweeping is
expensive. A thrown grenade and a bullet-like object get it; a crate does not. The machinery is in
[`PHMoveStorage.h`](PHMoveStorage.h.md), and the world's use of it in
[`PHWorld.cpp`](PHWorld.cpp.md).

## breaking

**`SplitProcess`** — hands off to the splitter holder and deletes the holder when no splitters
remain. The algorithm is in [`PHShellSplitter.cpp`](PHShellSplitter.cpp.md).

**`PassEndElements` / `PassEndJoints`** — move a contiguous range of elements or joints to another
shell: re-parent the first one, move its shapes between the two collision spaces, and re-point each
at the destination.

**`DeleteElement` / `DeleteJoint`** — deactivate and remove one by index.

**`BlockBreaking` / `UnblockBreaking`** — suspend and resume the break tests without discarding the
seams. Used while an object is being carried or scripted, where a break would be a bug.

## `SetJointRootGeom`

**Contract** — records, on a joint, which shape of its parent element the joint is anchored to —
specifically, the first shape of the parent's most recently opened seam. Does nothing if the parent
has no seams.

**Notes** — this is what lets [`PHFracture.cpp`](PHFracture.cpp.md) attribute a joint's reaction to
one side of a seam. Without it, a joint's force could not be assigned and every break test on a
jointed breakable would be wrong.

## dead code worth naming

**`StataticRootBonesCallBack`** and its getter are asserted-unreachable. **`SetCallbacksRecursive`**
asserts on entry. **`update_root_transforms`** returns the root element's frame when the animation
root and the physics root are the same bone and does nothing otherwise, with the interesting case
commented out — meaning a model whose animation root is not its physics root gets no correction.
**`ReanableObject`** is empty. **`get_animation_root_matrix`** returns its argument. **`PlaceBindToElForms`**
and **`BonesBindCalculate`** place elements and bones at their bind poses, and the first has a
precedence error in its condition that makes it skip a case it means to handle; it is called only
from a commented-out line in the splitter.

A rebuild should implement none of these. They are named here so a reader of the original does not
spend time on them.
