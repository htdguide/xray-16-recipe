# src/xrPhysics/PhysicsShell.cpp

> Turns a game object with a skeleton into a live physical shell — deciding what gets built,
> which bones are pinned, what the spawn configuration says, and refusing the ones that cannot
> work.

**Needs** — [`PhysicsShell.h`](PhysicsShell.h.md) · [`PHShell.h`](PHShell.h.md) · [`PHElement.h`](PHElement.h.md) · [`PHJoint.h`](PHJoint.h.md) · [`PHSplitedShell.h`](PHSplitedShell.h.md) · [`PHIsland.h`](PHIsland.h.md) · [`Physics.h`](Physics.h.md) · [`IPhysicsShellHolder.h`](IPhysicsShellHolder.h.md) · [`phvalide.h`](phvalide.h.md) · [`ExtendedGeom.h`](ExtendedGeom.h.md) · [`Include/xrRender/Kinematics.h`](../Include/xrRender/Kinematics.h.md) · [`xrCore/Animation/Bone.hpp`](../xrCore/Animation/Bone.hpp.md) · [Seam: Rigid-body dynamics](../../SYSTEM-REQUIREMENTS.md#seam-rigid-body-dynamics)
**Used by** — [`PhysicsShell.h`](PhysicsShell.h.md)
**Tier floor** — T2 for the build policy; T1 only where the static-environment callback
constructs a contact constraint by hand and hands it to the solver's island.

## Purpose

The abstract types in [`PhysicsShell.h`](PhysicsShell.h.md) say what a shell *is*; this file
says how one comes into existence. It is the single doorway between "a game object with a
model" and "a thing the solver simulates", and everything about that transition — what the
skeleton must contain, what the spawn configuration may override, what happens to pinned bones
— is decided here.

## Stateless, except for one scratch map.

The fixed-bone build path keeps one module-level map that it clears and refills per call. That
is a pooling decision, not a state decision, and it makes the function non-reentrant: two
threads cannot build shells at once. A rebuild should make it a local.

## `build_shell`

**Contract** — given a game object, whether to leave the shell inactive, an optional bone map
to fill in, and whether to skip installing bone callbacks, produce a fully built and activated
shell. Aborts (in a checked build) if the object's model is not a skeleton, has no physics
collision shapes, or sits at an invalid transform.

```text
FUNCTION build_shell(object, leave_inactive, bone_map, skip_bone_callbacks) -> Shell
  verify_object_model(object)                 # see below — refuses early and loudly
  skeleton := object.skeleton()

  shell := new Shell
  shell.build_from(skeleton, bone_map)        # creates elements and joints per bone
  shell.owner       := object
  shell.transform   := object.transform       # the shell starts where the object is
  shell.activate(leave_inactive, skip_bone_callbacks)
  shell.set_air_resistance()                  # defaults, always; never left unset
  RETURN shell
```

**Invariants** — the shell's transform is taken from the object, not the other way round.
This is the first half of the ownership contract: before activation the *object* owns the
placement; after activation the shell does.

**Notes** — air resistance is applied unconditionally with the module defaults rather than
left at the dynamics library's own default. Without it a light object dropped from height
accelerates without bound and the solver produces contacts deep enough to explode. The
constants are in [`PhysicsCommon.h`](PhysicsCommon.h.md).

## fixing bones

**Contract** — a caller may name a set of bones that must be *pinned*: their elements are
created and then fixed in place, so the rest of the shell hangs off them. Three surfaces
exist because callers hold the bone set in three different forms — a comma-separated text
list out of a configuration file, a list of bone ids, or a pre-seeded bone map — and all three
funnel into the same two-step shape:

```text
FUNCTION build_with_fixed_bones(object, leave_inactive, fixed_bone_names) -> Shell
  bone_map := empty
  FOR EACH name IN split(fixed_bone_names)
    id := skeleton.bone_id(name)
    FAIL WITH "wrong fixed bone" IF id is none     # a typo in data must not pass silently
    bone_map[id] := empty slot

  shell := build_shell(object, leave_inactive, bone_map)

  IF bone_map is non-empty
    shell.prefer_exact_integration()               # see note
  FOR EACH (id, slot) IN bone_map
    IF slot.element EXISTS
      slot.element.fix()
  RETURN shell
```

**Invariants** — the pin happens *after* the build, never during it, because an element must
exist and be placed before it can be pinned, and because the joints between it and its
neighbours must already be there or the chain hangs from nothing.

**Notes** — two things here are not obvious.

*Exact integration is requested whenever any bone is pinned.* A pinned body is an infinite
mass in the chain, which is exactly the configuration where the cheap integrator accumulates
error fastest; the more expensive integrator is the price of a hanging body that does not
slowly drift out of its socket.

*The strictness of the missing-element check differs by entry point.* Building from a text
list or a pre-seeded map treats "you named a bone that got no physics" as a hard error;
building from a list of bone ids skips it silently. The asymmetry is real and looks like
drift: the text path is fed by authored configuration, where a typo must be caught, while the
id path is fed by code that may legitimately offer bones it is not sure about. A rebuild
should make that distinction explicit rather than inferring it from the overload.

## `build_simple_shell`

**Contract** — produce a shell of exactly one element carrying one box: the model's own
bounding box, axis-aligned in the model's frame, with a caller-supplied mass. Activated from
the object's transform unless the object is attached to a parent, in which case the placement
is left to whoever owns the parent.

**Notes** — this is the fallback for objects that have a model but no authored collision
shapes: loot, thrown items, debris. The whole point is that it needs no skeleton data at all,
so the `verify` gate above does not apply and must not be called.

## applying the spawn configuration

**Contract** — a spawned entity may carry a small configuration block that retunes its shell.
Reading it is a policy decision list, not an algorithm:

```text
FUNCTION apply_spawn_config(config, shell, already_fixed)
  IF no config THEN RETURN

  IF config has section "physics_common"
    fix the bones named by its "fixed_bones" key

  IF config has section "collide"
    IF "ignore_static" is present AND (the shell is fixed OR it is an animated object)
      shell.ignore_static_geometry()
    IF "small_object"            THEN shell.mark_small()
    IF "ignore_small_objects"    THEN shell.ignore_small()
    IF "ignore_ragdoll"          THEN shell.ignore_ragdolls()
    IF "ignore_animated_objects" THEN shell.ignore_animated()

  IF config has section "animated_object"
    shell.create_animator(config, "animated_object")   # see PhysicsShellAnimator.cpp
```

**Invariants** — the presence of the `animated_object` section is what *makes* a shell an
animated one; there is no separate flag. That is a data-format fact a rebuild must preserve,
because the shipped spawn files rely on it.

**Notes** — `ignore_static` is honoured only for a shell that is pinned or animated. The
reason is that an unpinned free body that ignores the world falls through it; the guard exists
so a careless configuration line cannot delete an object. The original carries a note that the
condition is too coarse — it treats "has fixed bones" as "is really immobile", which is not
the same thing — and that remains an open decision rather than a settled one.

## validation: `verify_object_model` and `can_create_shell`

**Contract** — the same four conditions, checked twice with different consequences:

1. the object's model is a skeleton;
2. that skeleton has at least one **visible** bone whose authored shape is a *physics* shape;
3. the object's transform is numerically valid;
4. the object's position is inside the world's sane bounds
   ([`phvalide.h`](phvalide.h.md)).

`verify_object_model` aborts with a message naming the object and its model. `can_create_shell`
answers yes/no and hands back the same message as text, for callers that must degrade rather
than die.

**Invariants** — a bone contributes physics only if it is *both* visible and carries a physics
shape. Invisible bones are the skeleton's helper bones and must never become collision, or
every creature in the game grows invisible limbs.

**Notes** — two entry points for one predicate is the right shape, not duplication: the
assertion form catches authoring errors during development at the site where the data is
wrong, and the predicate form lets the spawn path refuse an object and log it in a shipping
build. A rebuild should write the predicate once and have the assertion call it.

## `non_elastic_collision_energy`

**Contract** — given two elements and a contact normal (pointing from the second toward the
first), report the kinetic energy that a perfectly inelastic collision along that normal
would dissipate. Delegates to the shared formula in [`Physics.cpp`](Physics.cpp.md); this is
just the public, element-typed doorway to it.

## `static_environment_contact_callback`

**Contract** — the object-contact callback that turns a contact against static geometry into
a constraint the solver will honour, and then tells the caller *not* to generate the contact
in the normal way.

```text
FUNCTION static_environment_contact(inout accept, this_side_is_first, contact, mat_a, mat_b)
  constraint := new contact constraint from contact
  moving_shape := this_side_is_first ? contact.shape_a : contact.shape_b
  island_of(moving_shape).connect(constraint)     # so the island knows it is not asleep
  attach constraint between the moving shape's body and the world
  accept := false                                 # we have handled it; do not double-add
```

**Invariants** — the constraint must be registered with the moving body's *island* as well as
attached to it. The island is the connected-component bookkeeping that decides what may sleep
together ([`PHIsland.h`](PHIsland.h.md)); a constraint attached but not registered lets a body
fall asleep while still being pushed, which reads to a player as an object frozen half-inside
a wall.

**Notes** — the handedness branch exists because the contact does not say which of its two
shapes is the dynamic one; the flag does. Attaching the constraint with the world on the
correct side is what makes the wall immovable rather than making the body immovable.

## `destroy_shell`

**Contract** — deactivate the shell, then release it. The order is the whole content: a shell
released while still registered leaves the world holding a reference to freed memory, which is
the physics half of the
[destroyed-entity invariant](../../SYSTEM-REQUIREMENTS.md#6-conformance). The caller's handle
is cleared as part of the call, so there is no window in which a stale pointer is reachable.
