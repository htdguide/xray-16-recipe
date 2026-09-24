# src/xrPhysics/PhysicsShellAnimator.cpp

> Makes a rigid-body shell follow an animation: every frame, each controlled body is welded to
> the pose the skeleton says its bone should be in.

**Needs** — [`PhysicsShellAnimator.h`](PhysicsShellAnimator.h.md) · [`PhysicsShellAnimatorBoneData.h`](PhysicsShellAnimatorBoneData.h.md) · [`PHShell.h`](PHShell.h.md) · [`PHElement.h`](PHElement.h.md) · [`PHDynamicData.h`](PHDynamicData.h.md) · [`PHIsland.h`](PHIsland.h.md) · [`IPhysicsShellHolder.h`](IPhysicsShellHolder.h.md) · [`Include/xrRender/KinematicsAnimated.h`](../Include/xrRender/KinematicsAnimated.h.md) · [`xrCore/Animation/Bone.hpp`](../xrCore/Animation/Bone.hpp.md) · [Seam: Rigid-body dynamics](../../SYSTEM-REQUIREMENTS.md#seam-rigid-body-dynamics)
**Used by** — [`PhysicsShellAnimator.h`](PhysicsShellAnimator.h.md)
**Tier floor** — T1: the target pose is handed to the solver as its own matrix and quaternion
types, and the mass-centre offset is computed in the solver's frame.

## Purpose

An *animated object* is the third kind of physical thing in the game, alongside free rigid
bodies and character controllers. It is a machine, a door, a swinging sign — something whose
motion is authored as an animation but which must still push the player and be pushed back by
the world. This file is how those two requirements are reconciled: the bodies are real rigid
bodies in the solver, and an animation is expressed to the solver as a moving constraint
rather than as a placement.

The alternative — teleporting the bodies onto the animated pose each frame — is what a
rebuilder will reach for first, and it is wrong: a teleported body has no velocity, so it
passes through anything it touches and imparts no momentum. Welding is what makes an animated
crusher actually crush.

## State

```text
RECORD ShellAnimator
  bones       : list<ControlledBone>   # see PhysicsShellAnimatorBoneData
  shell       : Shell
  start_xform : matrix                 # the owning object's transform when the
                                       # animator was created — the frame the
                                       # animation's bone poses are interpreted in
```

**Invariants** — `start_xform` is captured once and never updated. Everything the animator
does is relative to it, which means the animated object's *root* does not move: the animation
moves parts within a fixed frame. An object that must also translate needs its root driven by
something else.

## construction

**Contract** — from a shell and a configuration section, decide which bodies are controlled,
weld each to the world, and then decide what happens to the shell's own joints.

```text
FUNCTION create_animator(shell, config, section)
  start_xform := shell.owner.transform

  controlled := config[section]."controled_bones"        # note the shipped spelling
  IF controlled is absent OR controlled == "all"
    FOR EACH element IN shell.elements  -> weld(element)
  ELSE
    FOR EACH name IN split(controlled)
      id := shell.skeleton.bone_id(name)
      FAIL WITH "controled bone not found" IF id is none
      FAIL WITH "controled bone has no physics" IF shell.element_of(id) is none
      weld(shell.element_of(id))

  IF config[section]."leave_joints" == "all"
    RETURN                                               # keep the authored joints
  delete every joint in the shell                        # otherwise: welds only
```

```text
FUNCTION weld(element) -> ControlledBone
  constraint := new fixed constraint
  shell.active_island.register(constraint)
  attach constraint between element.body and the world
  set the constraint's target to the bodies' CURRENT relative pose
  RETURN (element, constraint)
```

**Invariants** — the constraint is registered with the shell's active island *and* attached.
Registration is what keeps the island awake and accounted for; attachment is what makes the
solver honour it. One without the other is a body that either sleeps while being driven or is
driven while the island thinks it is idle.

**Notes** — the default when `controled_bones` is absent is *all* bodies, and deleting the
shell's own joints is the default too. Both defaults say the same thing: an animated object is
normally rigid, fully authored, and its authored joint chain is redundant once every part is
welded to the animation. `leave_joints = all` is the escape hatch for a hybrid — a door whose
hinge is real but whose handle is animated — and it is the only configuration in which joints
and welds coexist.

The key name in the shipped data is misspelled (`controled_bones`). A rebuild must accept the
misspelling, because [the configuration is frozen on the read side](../../SYSTEM-REQUIREMENTS.md#5-data-and-persistence).

## destruction

**Contract** — unregister every weld from the island, then destroy it. Both, in that order;
destroying a constraint the island still lists is a dangling reference inside the solver.

## `OnFrame`

**Contract** — once per frame, re-aim every weld at where animation now says its bone is.

```text
FUNCTION on_frame()
  shell.enable()                          # an animated shell must never fall asleep

  FOR EACH bone IN bones
    instance := skeleton.bone_instance(bone.element.bone_id)
    instance.clear_callback()             # hand bone ownership BACK to animation
    skeleton.invalidate_bones()
    skeleton.evaluate_bones(force := true)

    target := start_xform COMPOSED WITH instance.transform
    target_centre := bone.element.mass_centre_under(target)
    bone.constraint.set_target(orientation_of(target), target_centre)
```

**Invariants** — the weld's target is the **mass centre** under the target transform, not the
transform's own origin. A body's origin and its centre of mass coincide by construction
([`Geometry.cpp`](Geometry.cpp.md)), so anchoring at the origin would offset every animated
part by the difference and the object would visibly shear.

**Notes** — three things here are worth naming.

*The bone callback is cleared, not installed.* This is the exact inverse of every other
physical object, and it is the ownership contract read backwards: for an animated object the
**animation owns the bone transform** and physics follows it, so the shell must not be driving
the skeleton. A shell that both welds and installs bone callbacks chases its own tail and the
object vibrates.

*The skeleton is invalidated and re-evaluated inside the loop, once per controlled bone.* That
is redundant — the pose does not change between bones — and it is the single most expensive
line in the file for an object with many parts. A rebuild should evaluate once before the
loop; nothing depends on the repetition.

*The shell is woken unconditionally every frame.* An animated object has no rest state as far
as the sleeping heuristic is concerned: its bodies may be perfectly still while the animation
is about to move them, and a sleeping body ignores a moved weld. The cost is that an animated
object never stops consuming solver time, which is why the `animated_object` section is opt-in
per spawn rather than a property of a model.
