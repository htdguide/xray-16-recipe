# src/xrPhysics/PhysicsShell.h

> The four abstract types every physical object in the game is built from — a transformable
> thing, a rigid element, a joint and a shell — and the contract each demands of its
> implementor.

**Needs** — [`PHDefs.h`](PHDefs.h.md) · [`PhysicsCommon.h`](PhysicsCommon.h.md) · [`ICollideValidator.h`](ICollideValidator.h.md) · [`PhysicsExternalCommon.h`](PhysicsExternalCommon.h.md) · [`IPhysicsShellHolder.h`](IPhysicsShellHolder.h.md) · [`PhysicsShell.cpp`](PhysicsShell.cpp.md) · [`xrEngine/IPhysicsShell.h`](../xrEngine/IPhysicsShell.h.md) · [Seam: Rigid-body dynamics](../../SYSTEM-REQUIREMENTS.md#seam-rigid-body-dynamics) · [Seam: Script binding layer](../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer)
**Used by** — [`BastArtifact.cpp`](../xrGame/BastArtifact.cpp.md) · [`Bolt.cpp`](../xrGame/Bolt.cpp.md) · [`BreakableObject.cpp`](../xrGame/BreakableObject.cpp.md) · [`CaptureBoneCallback.h`](../xrGame/CaptureBoneCallback.h.md) · [`Car.cpp`](../xrGame/Car.cpp.md) · [`Car.h`](../xrGame/Car.h.md) · [`CarWeapon.cpp`](../xrGame/CarWeapon.cpp.md) · [`CustomRocket.cpp`](../xrGame/CustomRocket.cpp.md) · [`Explosive.cpp`](../xrGame/Explosive.cpp.md) · [`ExplosiveRocket.cpp`](../xrGame/ExplosiveRocket.cpp.md) · [`Grenade.cpp`](../xrGame/Grenade.cpp.md) · [`HangingLamp.cpp`](../xrGame/HangingLamp.cpp.md) · [`Helicopter.cpp`](../xrGame/Helicopter.cpp.md) · [`HolderEntityObject.cpp`](../xrGame/HolderEntityObject.cpp.md) · _and 36 more_
**Tier floor** — T2 as written: these are interfaces. The mass tensor passed by reference and
the raw contact record leak the dynamics library's layout through two methods, and those two
are what pin the *implementations* to T1.

## Purpose

This is the widest interface in the chapter and the one a rebuild must get right first,
because every other decision hangs off the shape of these four types. It declares:

- **a transformable physical thing** — anything with a placement, a mass, an extent, and the
  ability to be pushed;
- **an element** — one rigid body made of one or more primitive shapes;
- **a joint** — a constraint between two elements;
- **a shell** — the set of elements and joints built from one skeleton, which is what a game
  object actually owns.

The layering is not arbitrary: a shell *is* a transformable thing, so a caller that only
wants to move something or push it does not need to know whether it is one body or forty.
That is what lets the same code shove a crate, a corpse and a car.

The header also declares the free functions that create these things, which live in
[`PhysicsShell.cpp`](PhysicsShell.cpp.md).

## State

```text
ENUM motion_history_state
  clear          # this thing teleported: forget where it was, do not sweep from there
  unspecified    # placement happened, but the caller has no opinion
  not_clear      # this thing moved continuously: keep the swept history

RECORD physicsBone                 # what a bone maps to, once a shell is built
  joint   : optional<Joint>
  element : optional<Element>
# a build call is given a map keyed by bone id whose entries it FILLS IN; the
# caller pre-seeds the keys it cares about (fixed bones) and reads back what
# physics was created for them
```

## `CPhysicsBase` — a transformable physical thing

**Contract** — the common surface of an element and a shell. An implementor must provide:

- **A transform in the parent's space**, readable and writable. This is the field the render
  side and the game side both look at, and the ownership rule below governs who may write it.
- **Four ways to become active**, and the difference between them is load-bearing:
  1. *from a motion* — given the transform at the start of the previous frame, a fraction
     along it, and the transform now: the body starts moving with the velocity implied by
     that motion. This is how a thrown or dropped object inherits the hand's or the carrier's
     motion instead of appearing at rest.
  2. *from a transform plus explicit linear and angular velocity* — the network and the
     script path.
  3. *from nothing* — take the placement already in the transform field, at rest.
  4. *from a transform* — place there, at rest.
  Each takes a flag asking for the body to start **disabled** (asleep), and the bare form
  additionally takes a flag asking that bone callbacks *not* be installed.
- **Interpolated read-back** of the global transform and position, for rendering. The
  simulation runs on a fixed step and the frame does not; the renderer must read an
  interpolated pose, never the raw one, or objects visibly stutter
  ([`PHInterpolation.h`](PHInterpolation.h.md)).
- **Network import and export** of its dynamic state.
- **Status queries**: breakable, enabled (awake), active, *fully* active, and the ability to
  deactivate and to wake.
- **Mass, density and volume**, settable and gettable; setting mass and setting density are
  different operations, because density derives mass from the authored shapes' volume.
- **Extent along an arbitrary axis**, from which the oriented box is built.
- **Force and impulse application**: force by direction and magnitude or by components,
  impulse likewise, torque, a directly-set force, and a gravity acceleration applied as a
  body force rather than by the world.
- **Air resistance** (a linear and an angular damping coefficient) and **dynamic limits**
  (caps on linear and angular speed) with **dynamic scales** (the fraction by which an
  over-limit velocity is scaled back). Defaults come from
  [`PhysicsCommon.h`](PhysicsCommon.h.md).
- **Callback wiring**: set the static-contact callback, set *or* add *or* remove an
  object-contact callback, set opaque callback data, set the owning game object, and mark
  the thing as animated.
- **Placement with motion history**: setting a transform always says whether the move was a
  teleport or continuous motion. There is no form that leaves this unsaid, because a
  placement whose history is wrong either snags the object on the walls it flew through or
  lets it tunnel through the floor on its next step.
- **Material assignment** by index or by name, and per-object sleeping parameters.

**Invariants** — *whoever is active owns the transform.* While a thing is active, the physics
step writes its transform and the game object reads it; while it is not, the game object
writes and physics reads it on activation. Every bug in this chapter that looks like an object
teleporting, vibrating or sinking is a violation of that one sentence. `isFullActive` exists
because there is a third state — registered and simulated but not yet driving the visual —
and code that only checks `isActive` gets that case wrong.

**Notes** — the `setForce`/`applyForce` pair is not redundancy: one *replaces* the
accumulated force for the step, the other adds to it. Similarly the object-contact callback
has both a setter (this shape's behaviour) and an adder (another subsystem also watching);
confusing them silently drops a subsystem's hook. See
[`Geometry.cpp`](Geometry.cpp.md), which carries the same distinction one level down.

## `CPhysicsElement` — one rigid body

**Contract** — a rigid element is a single body carrying one or more primitive shapes. Beyond
the common surface it must provide:

- **Shape construction**: add a sphere, a box, a cylinder, or a shape read straight from the
  skeleton's bone data, with or without an extra offset. Shapes are enumerable and removable,
  and the element knows whether it has any at all.
- **Mass construction**: set the tensor, add to it, set mass or density *about a given mass
  centre*, set a box's mass, and read the tensor back. The centre of mass is exposed both in
  world and in the element's own frame, and can be moved explicitly.
- **Impulse application in three frames** — relative to the mass centre, relative to the
  element's geometric frame, and *traced*: an impulse at a point with a bone id, which is
  what a bullet hit produces. The three exist because the caller knows a position in
  different spaces depending on where it came from, and converting at the call site is where
  sign errors live.
- **Fracture registration**: a shape may be marked fracturable, which returns a handle used
  later when the element actually splits ([`PHFracture.h`](PHFracture.h.md)).
- **Fixing**: an element can be pinned in place and released again, and reports whether it is
  pinned. This is how authored "fixed bones" turn a ragdoll into a hanging body or a door
  into a hinged one.
- **Parent element** and **self id** (the bone id it was built from), so a shell can walk
  its tree.
- **Point velocity** at an arbitrary world point, and the direction of its largest projected
  area — the latter used to decide which way an explosion should tumble a flat object.
- **Radius**, and the dynamic global transform read straight from the solver.

**Notes** — `setMassMC`/`setDensityMC` take a mass centre because a body's origin and its
centre of mass are made to coincide at build time
([`Geometry.cpp`](Geometry.cpp.md)); an element that later gains or loses shapes must
re-establish that, and these are the entry points that do it.

## `CPhysicsJoint` — a constraint between two elements

**Contract** — a joint of one of five kinds:

```text
ENUM joint_kind
  ball           # 0 axes of control, 3 rotational degrees free
  hinge          # 1 axis
  hinge2         # 2 axes — the wheel joint: steering axis plus spin axis
  full_control   # 3 axes, addressed as Euler angles — the ragdoll joint
  slider         # 1 translational axis
```

Every setter that names a direction or a point comes in three coordinate systems — the first
element's local frame, the second element's local frame, and the world — and the joint
remembers which one was used so it can report the axis back in the same terms. That triplication
is the interface's largest single cost and it is load-bearing: a ragdoll's limits are authored
in the *bone's* frame, a vehicle's steering axis in the *chassis'* frame, and a script sets
things in world terms.

An implementor must provide:

- **Anchor and axes**, in any of the three frames, per axis index.
- **Limits** (low and high stop) per axis, settable statically or *dynamically* — one stop at
  a time, mid-simulation, which is how a door is locked, a ladder step is gated and a ragdoll
  joint is stiffened during a hit animation.
- **Spring and damping factors**, per axis and for the joint as a whole, plus a "fudge
  factor" applied while the joint is active. These are the three numbers the shipped ragdoll
  and vehicle tuning data is expressed in, and
  [the seam note](../../SYSTEM-REQUIREMENTS.md#seam-rigid-body-dynamics) warns that they are
  the part that transfers least cleanly to another dynamics library.
- **Motors**: a maximum force and a target velocity, per axis or for all axes, both settable
  and readable. This is the whole steering-and-engine surface a vehicle needs.
- **Breakability**: a force and a torque threshold, plus a destruction descriptor consulted
  when the threshold is crossed ([`PHJointDestroyInfo.h`](PHJointDestroyInfo.h.md)).
- **Readouts**: current angle and angular rate per axis, current axis direction and anchor in
  world terms, and the number of axes.
- **Lifecycle**: create (build the constraint), activate, run one simulation step's worth of
  its own bookkeeping, deactivate — and a back-reference slot so the owner can be told when
  the joint destroys itself.

**Notes** — `IsWheelJoint` / `IsHingeJoint` exist because the vehicle and the ragdoll code
each need to know they are holding their own kind before reaching for axis semantics that
only that kind has. In a rebuild with a sum type over joint kinds these disappear.

## `CPhysicsShell` — a skeleton's worth of physics

**Contract** — a shell owns a list of elements and a list of joints built from one skeleton,
plus the skeleton pointer itself. It is what a game object holds. An implementor must provide:

**Construction.** Build from a skeleton, in two phases — a *pre*-build that creates the
elements and joints without activating anything, and a build that finishes the job — plus a
plain `Build`. The two-phase form exists because a caller must be able to reach in between and
fix bones, set masses or delete joints before the shell ever steps. The build call also takes
the bone map described above, filled in with what was created for each requested bone.
`ActivatingBonePoses` snapshots the skeleton's current pose as the shell's starting placement.

**Lookup.** Element and joint by bone id, by bone name (shared string or plain text), and by
*store order* — the index in the shell's own list, which is not the bone id and is the key the
network layer and the script layer both use, because it is stable and dense. A shell also
answers the geometry that belongs to a bone, and the nearest element to a world point (with an
optional filter, so a caller can restrict the search to, say, unbroken elements).

**Collision classification.** A shell registers itself into a *collision group* and carries
two bit sets: which classes it collides with, and which class it belongs to. The named
switches — ignore static geometry, ignore dynamic objects, mark as ragdoll, ignore ragdolls,
mark as small, ignore small objects, ignore animated objects — are how the shipped
configuration keeps corpses from shoving crates and grass from fighting the player. See
[`ICollideValidator.h`](ICollideValidator.h.md) for the pairwise rule these bits feed.

**The bone-callback contract.** This is the ownership handoff, and it is the shell's most
important responsibility. When a shell is active it installs a callback on each bone instance
it drives; the skeleton's evaluation then asks physics for that bone's transform instead of
computing it from animation. The surface is: set the callbacks, clear them all, reset a
subset by bone id and mask, enable or disable them wholesale, and choose whether the callback
*overwrites* the animated transform or composes with it. Two distinct callbacks exist — one
for a live shell and one for a shell attached to a static object — because a static object's
bones must not be re-evaluated per frame.

- `ToAnimBonesPositions` drives the bodies *to* where animation says the bones are: the
  reverse direction, used when a ragdoll is being blended back into an animation.
- `AnimToVelocityState` converts the difference between the animated pose and the physical
  pose into velocities, clamped by a linear and an angular limit, and reports whether the two
  are now close enough to consider the blend finished. That boolean is the exit condition of
  every animation-to-ragdoll transition in the game.

**Stepping and freezing.** A shell can be frozen and unfrozen, disabled (put to sleep), have
its collision disabled and re-enabled, have character collision specifically removed, and be
stepped *on its own* (`PureStep`) outside the world's timeline — which is how the activation
procedures push a freshly spawned body out of a wall. `CollideAll` forces a full collision
pass. `SetGlTransformDynamic` moves the whole shell as a unit while it is simulating.

**Breaking and splitting.** A shell knows whether it has fractured, holds a splitter that
decides where it may come apart, can be told to block and unblock breaking (used while a
scripted sequence runs), and can perform the split, yielding the pairs of new shells that
result.

**Hits.** An impulse traced to a point, optionally with a bone id, and a full hit carrying a
hit *type* — the type matters because an explosion is not applied as a single impulse
(see [`ShellHit.cpp`](ShellHit.cpp.md)).

**Object-in-root bookkeeping.** A shell keeps the game object's transform expressed in the
root element's frame, and can be told to recompute it. This is what lets the object's visual
origin differ from the root body's origin — a car's model origin is not its chassis' centre
of mass — and `UpdateRoot` re-derives the object transform from the root body after a step.

**Traced geometry.** Individual shapes can be marked as "traced", which makes the collider
keep per-shape swept history for them; the set can be added to, cleared, enabled and disabled
wholesale, and queried. This costs memory per shape and is therefore opt-in.

**Exact integration.** A shell can ask for the more expensive, more accurate integrator. The
build path turns this on automatically whenever any bone is fixed, because a pinned body in a
chain is exactly the case where the cheap integrator drifts.

**Invariants** — a shell's element list order is its store order and must not be permuted
after build: the network protocol and saved games index into it. Every element in a shell
belongs to one collision group with the shell, so that a ragdoll's limbs do not collide with
each other unless the configuration says they may.

## `StaticEnvironmentCB`

**Contract** — declared here, implemented in [`PhysicsShell.cpp`](PhysicsShell.cpp.md).

## creation and destruction entry points

**Contract** — declared here, implemented in [`PhysicsShell.cpp`](PhysicsShell.cpp.md):
create a bare element, shell, splittable shell or joint; build a shell from a game object
(with fixed bones named in text, as bone ids, or not at all); build a one-box simple shell;
apply a spawn configuration section to a built shell; destroy a shell; and the two validators
that answer whether a given object *can* have a shell and why not.

## `NearestToPointCallback`

**Contract** — a filter the nearest-element search consults: it is offered each candidate
element and answers whether that element is eligible. Exists so a caller can exclude broken,
fixed or already-hit elements without the search knowing what those words mean.

## `shape_is_physic` / `has_physics_collision_shapes`

**Contract** — declared here, implemented in [`PhysicsShell.cpp`](PhysicsShell.cpp.md).
