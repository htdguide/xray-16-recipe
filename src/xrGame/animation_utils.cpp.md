# src/xrGame/animation_utils.cpp

> Pins one bone to a fixed offset from its parent regardless of what the animation says, and answers whether one bone is an ancestor of another.

**Needs** — [`animation_utils.h`](animation_utils.h.md) · [`Include/xrRender/Kinematics.h`](../Include/xrRender/Kinematics.h.md) · [`xrCore/Animation/Bone.hpp`](../xrCore/Animation/Bone.hpp.md) · [`game_object_space.h`](game_object_space.h.md)
**Used by** — [`animation_utils.h`](animation_utils.h.md)
**Tier floor** — T2: transform algebra inside the pose evaluation; must not allocate there

## Purpose

Two unrelated helpers that both reach into the skeleton.

The first freezes a bone. When part of a creature is taken over by something other than its
animation — a limb under physics control, a head locked by an aiming solver, a bone whose
sub-tree has been handed to a ragdoll — the animation must stop writing that bone, but the
rest of the skeleton must keep animating around it. The fix captures the bone's current
offset from its parent and reasserts that offset every time the pose is evaluated. The bone
then rides its parent rigidly: it follows the body but stops moving on its own.

The second is a pure query on the bone hierarchy, used wherever a decision must apply to a
whole sub-tree — whether a hit landed anywhere below the pelvis, whether a physics shape
covers a given bone.

The file is a grab-bag; the two have nothing to do with each other beyond both touching
the skeleton, and a rebuild may put either anywhere.

## State

```text
RECORD anim_bone_fix
  bone    : optional<reference to a bone instance>   # none == not installed
  parent  : optional<reference to its parent bone instance>
  matrix  : matrix     # the frozen child-relative-to-parent transform
```

**Invariants** — `bone` and `parent` are both present or both absent; the fix is installed
exactly when they are present. Installing requires the bone to have no other callback (the
callback slot is single-occupancy) and the bone not to be the root, which has no parent to
ride. The record must be released before it is destroyed — a destroyed fix whose callback is
still installed leaves the pose evaluation calling into freed memory, which is why the
destructor asserts rather than cleaning up: a fix outliving its release point is a bug in
the caller, not something to paper over.

## `fix`

**Contract** — installs the freeze on the named bone of the given model. Captures the
bone's *current* transform expressed in its parent's frame, so the bone freezes wherever it
happens to be at the moment of the call — the caller controls the frozen pose by choosing
when to call. Allocates nothing.

```text
FUNCTION fix(bone_id, model)
  REQUIRE bone_id is not the root bone
  b = model.bone_instance(bone_id)
  REQUIRE b has no callback
  bone   = b
  parent = model.bone_instance(model.bone_data(bone_id).parent)
  matrix = inverse(parent.transform) COMPOSED WITH b.transform   # capture the offset now
  b.set_callback(custom, reassert, self)
```

## the reassertion callback

**Contract** — runs inside pose evaluation, after the parent's matrix is final and before
any child's is computed. Overwrites the bone's world transform with parent-times-frozen-
offset, discarding whatever the animation produced for it.

```text
FUNCTION reassert(bone_instance)
  bone_instance.transform = parent.transform COMPOSED WITH matrix
```

**Notes** — the ordering requirement is the whole contract: the pose evaluation must
already visit bones parent-before-child for this to be correct, and it does. A rebuild that
evaluates bones in any other order cannot express this as a per-bone hook.

**Notes** — the result is validated for finiteness before it is used. A non-finite bone
transform propagates to every descendant and then into the skinning, and the resulting
model is not merely wrong but unrecoverable, so it is caught at the source.

## `refix`

**Contract** — reinstalls the callback on the already-known bone without recapturing the
offset. Used after something has cleared every bone callback on the model wholesale (a
visual change, a reset) and the freeze must come back with its original frozen pose rather
than with whatever the skeleton has drifted to since.

## `release` and `deinit`

**Contract** — `release` removes the callback, asserting that the callback found there is
still this fix's; `deinit` additionally forgets the bone and parent, returning the record
to its uninstalled state so it can be installed again. Neither allocates. A fix released
but not deinitialized still remembers its bone, which is what makes `refix` possible.

## `find_in_parents`

**Contract** — true if the sought bone lies on the chain from the given bone up to (but not
including) the root. Walks parent links, terminating at the root or at a missing parent.
Pure; touches no state.

```text
FUNCTION find_in_parents(sought, from, model) -> bool
  b = from
  WHILE b is not the root bone AND b exists
    IF b == sought THEN RETURN true
    b = model.bone_data(b).parent
  RETURN false
```

**Notes** — the root is excluded from the search, so "is this bone under the root" is always
false. That is deliberate — every bone is under the root, so the question is never worth
asking — but it makes the function asymmetric with its name, and a rebuilder should keep the
exclusion rather than fix it: callers rely on it to mean "under this *sub-tree*, which is
not the whole skeleton".
