# src/xrPhysics/PHShellActivate.cpp

> The five ways a shell starts simulating, the one way it stops, and the ordering constraints that make the transition between animation and physics invisible.

**Needs** — [`PHShell.h`](PHShell.h.md) · [`PHElement.h`](PHElement.h.md) · [`PHElementInline.h`](PHElementInline.h.md) · [`PHJoint.h`](PHJoint.h.md) · [`PHWorld.h`](PHWorld.h.md) · [`PHObject.h`](PHObject.h.md) · [`PHCollideValidator.h`](PHCollideValidator.h.md) · [`PhysicsShellAnimator.h`](PhysicsShellAnimator.h.md) · [`IPhysicsShellHolder.h`](IPhysicsShellHolder.h.md) · [`Include/xrRender/Kinematics.h`](../Include/xrRender/Kinematics.h.md) · [Seam: Rigid-body dynamics](../../SYSTEM-REQUIREMENTS.md#seam-rigid-body-dynamics)
**Used by** — [`PHShell.h`](PHShell.h.md)
**Tier floor** — T1: it creates and destroys collision spaces and solver bodies in a defined order.

## Purpose

Activation is where the chapter's ownership contract is actually exercised, and it has more shapes
than it looks like it should because the callers know different amounts. A door knows its exact
placement. A thrown object knows where it was and where it is, and wants the implied velocity. A
corpse becoming a ragdoll knows nothing except that the animation system has the pose. Each gets its
own entry point.

## the shared prologue

**Contract** — every activation begins the same way.

```text
FUNCTION activate(start_disabled)
  preset_active()                      # create the collision space if it does not exist
  IF this object is not yet in the world's active set THEN vis_update_deactivate()
  IF NOT start_disabled THEN enable_object()
```

**Notes** — `PresetActive` creates the shell's collision space, with the space's own cleanup switched
off because the shell owns its shapes and will destroy them itself. Creating the space here rather
than at construction means a shell that is built and never activated costs nothing.

The `vis_update_deactivate` call is a counter decrement that pairs with an increment on deactivation;
its purpose is to let the game object know when nothing is asking it to keep processing. Calling it
from *activation* reads backwards and is: a shell that starts simulating no longer needs the game
object's own per-frame processing, because physics is driving it now.

## `Activate(transform)` — the plain form

**Contract** — place the shell at a transform, build and start every element and joint, install the
bone callbacks, register the shell spatially, and mark it active-but-activating.

```text
FUNCTION activate(transform, start_disabled, skip_bone_callbacks)
  IF already active THEN RETURN
  activate(start_disabled)
  FOR EACH element  → activate it at the shell's frame
  FOR EACH joint    → activate it
  IF a skeleton exists AND callbacks are wanted THEN set_callbacks()
  register spatially
  active := true ; activating := true
```

**Invariants** — **elements before joints, always.** A joint resolves its anchor and axes against its
two bodies' current placements at creation (see [`PHJoint.cpp`](PHJoint.cpp.md)), so the bodies must
already be where the authored data expects them. Reversing the order gives a shell whose joints are
anchored at the origin.

**Notes** — the `skip_bone_callbacks` variant temporarily hides the skeleton from the elements while
they activate, so no element installs a callback on its bone. That is how a shell is started as a
pure simulation with no visual binding — the caller intends to read its transforms rather than let
it drive a model.

`activating` is set and not cleared here. It is cleared by the first bone evaluation, or by the next
frame update, whichever comes first. Until then the shell is `isActive` but not `isFullActive`, and
the operations that need a settled placement check the latter.

## `Activate(from, fraction, to)` — the implied-velocity form

**Contract** — place the shell at the first transform and give it the linear velocity implied by the
straight line to the second. The fraction argument is ignored.

```text
FUNCTION activate(m0, dt01, m2, start_disabled)
  … the plain activation at m0 …
  # correct for the fact that the elements placed themselves from their bones, not from m0
  actual := get_global_transform_dynamic()
  correction := m0 · inverse(actual)
  transform_position(correction)
  set_linear_velocity(m2.origin - m0.origin)
```

**Notes** — two things here.

*The correction pass is the interesting part.* Each element placed itself at the shell's frame
composed with its own bind pose, so the shell's *reported* transform afterwards — which goes through
the root's mass centre and the object-in-root bridge — is not the transform that was asked for. The
correction measures the discrepancy and applies it to every element. Any rebuild that places a
multi-body shell from an object-space transform needs the same reconciliation, or every object
activates slightly offset from where it was asked to be.

*The velocity is a difference of origins with no divisor.* It is a displacement, not a velocity, and
the fraction argument that would have converted it is ignored. Angular velocity is not derived at
all, so an object that was spinning when it became physical stops spinning. Both are reproduced
behaviour: the shipped tuning of thrown objects is calibrated against it.

## `Activate(transform, linear, angular)` — the explicit form

**Contract** — the plain activation with both velocities given, applied per element at activation
rather than afterwards. No correction pass: the caller is taken at its word.

## `Build` and `RunSimulation` — activation in two halves

**Contract** — `Build` creates every element's body and every joint's constraint, and marks the shell
active, **without** adding anything to the world's simulation. `RunSimulation` then adds the shapes
to the collision space, the bodies and constraints to the island, and registers the shell spatially.

**Notes** — the split exists so a caller can construct a shell, adjust it — set masses, add shapes,
move elements — and only then commit it to the simulation. Between the two calls the shell reports
itself active but the solver has never seen it. That is a real state a rebuild must have, because
mass assembly and joint construction both need the bodies to exist.

`RunSimulation` takes a flag to skip re-placing the elements, for the case where they have already
been placed by hand and re-placing them would undo the adjustment.

## `PureActivate` and `AfterSetActive` — the split-product path

**Contract** — `PureActivate` marks a shell active and registers it *without building anything*: its
elements and joints already exist, because they were moved in from another shell. It resets the
object-in-root bridge to identity. `AfterSetActive` is `PureActivate` plus telling every element to
preset itself active.

**Invariants** — resetting the object-in-root bridge to identity is correct for a fragment and only
for a fragment: a piece that broke off has no authored object origin of its own, so its origin *is*
its root body's. A rebuild must not generalize this.

**Notes** — these two are how [`PHShellSplitter.cpp`](PHShellSplitter.cpp.md) brings a new shell to
life mid-step without ever letting it pass through an inactive state that the solver might observe.

## `Deactivate`

**Contract** — tear a shell down completely, in a defined order, refusing to do it at a moment when
the world could observe a partially-destroyed object.

```text
FUNCTION deactivate()
  tell the world to release every reference it holds to this shell
  IF an animator exists THEN stop the object's processing and destroy the animator
  IF NOT active THEN RETURN

  REQUIRE the world is not mid-step
  REQUIRE neither the world nor this object is frozen

  zero_callbacks()                       # animation takes its bones back FIRST
  IF fully active THEN
    vis_update_deactivate()
    # force one contact-resolution pass with only this object live, so that everything
    # touching it learns it is going away
    activate this object ; freeze the world ; unfreeze this object
    step the touch pass ; unfreeze the world
  unregister spatially
  vis_update_activate()                  # the game object must process itself again
  disable_object()
  remove from the recently-deactivated list
  FOR EACH element → deactivate
  FOR EACH joint   → deactivate
  destroy the collision space
  active := false ; activating := false
  clear the swept-shape list
```

**Invariants** — the ordering is the content of this procedure and every step of it is load-bearing:

- **Callbacks go first.** A bone whose owning element is destroyed but whose callback is still
  installed is a dangling reference the very next time the skeleton is evaluated.
- **The world must not be mid-step.** The solver holds pointers to these bodies for the duration of a
  step; this is asserted rather than handled, and a rebuild should make it structurally impossible
  instead.
- **A frozen object cannot be deactivated.** Freezing captures state to be restored; destroying a
  frozen object leaves that capture pointing at nothing.
- **Elements before joints** — the reverse of activation. A joint's constraints must be removed from
  the island before its bodies are, which is why elements' shapes leave first and then the joints
  destroy their constraints; see the ordering note in [`PHJoint.cpp`](PHJoint.cpp.md).
- **The forced touch pass** is the subtle one. Objects resting on this one, or constrained to it, hold
  contacts the solver established on a previous step. Freezing everything else and running one
  collision pass with only this object live makes each of them re-evaluate, so they wake up and fall
  instead of hanging in the air where the vanished object used to hold them.

## `ActivatingBonePoses`

**Contract** — force every element to adopt its bone's *current animated* pose immediately, rather
than waiting for the skeleton's own evaluation.

**Notes** — the ordinary handover is deferred to the first bone callback, because that is the first
moment the animated pose is known to be current (see [`PHElementInline.h`](PHElementInline.h.md)).
This is the override for callers that have just evaluated the skeleton themselves and know the pose
is fresh — a death animation reaching its final frame, handing off to a ragdoll in the same tick.
It is the difference between a ragdoll that starts from the pose the player saw and one that starts
from the pose of a frame ago.
