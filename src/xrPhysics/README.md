# src/xrPhysics

> Chapter 16 — rigid bodies, joints, ragdolls, vehicles, and the bridge between the dynamics
> library and the engine's world.

## What this module is responsible for

The constraint solver is **not** in this chapter. It is a
[seam marked *given*](../../SYSTEM-REQUIREMENTS.md#seam-rigid-body-dynamics): bodies with
mass and inertia, six kinds of joint with stops and motors, contact parameters, and a step
that can be interrupted between collision detection and the constraint solve. Shop for one.

This chapter is everything around it, and that is where the game-defining decisions live.

A solver knows about bodies. The game knows about *objects with skeletons*, which are
sometimes animated, sometimes simulated and frequently both at once. Turning one into the
other — deciding which bones get bodies, which pairs get joints, what mass each carries, who
owns a bone's transform at which moment, and how control is handed over without a visible
snap — is this module's real subject. So is the character controller, which is not a rigid
body at all and only pretends to be one. So is the triangle-mesh collider that answers the
solver from the engine's own collision database. So is every rule that keeps the shipped
content from exploding: sleep thresholds, velocity clamps, non-finite recovery, collision
filtering, and the activation procedures that find a legal place for a volume before
anything is created there.

The module is also the engine's most dangerous, in a specific sense. It is the only place
where a single bad number propagates: a non-finite position on one body reaches every body
it touches within a few steps, and the world does not recover. A large fraction of the code
here is defence against that, and a rebuilder who removes it as noise will ship a game that
works until it does not.

## Where it sits in the build order

Chapter 16, after [`src/xrUICore`](../xrUICore/README.md) and before
[`src/Layers/xrRender`](../Layers/xrRender/README.md). It rests on:

- [`src/xrCore`](../xrCore/README.md) — containers, the configuration format, the animation
  data model (a skeleton and its bones come from there).
- [`src/xrCDB`](../xrCDB/README.md) — the static collision database. The level is queried,
  never copied into the solver.
- [`src/xrMaterialSystem`](../xrMaterialSystem/README.md) — surface materials and the
  pairwise interaction table. Every contact's friction, bounce and softness is derived from
  the *pair* of materials involved, and several behaviours are keyed to per-material flags.
- [`src/utils/xrMiscMath`](../utils/xrMiscMath/README.md) — vectors, matrices, quaternions.
- [`src/xrScriptEngine`](../xrScriptEngine/README.md) — parts of the module are exported to
  Lua, and the exported names are frozen by
  [criterion 10](../../SYSTEM-REQUIREMENTS.md#6-conformance).

It also links back against [`src/xrEngine`](../xrEngine/README.md), which is one of the three
declared cycles: the engine owns the frame loop and declares the ports (`IPhysicsShell`,
`IPhysicsShellHolder`, `IPHWorld`), and this module provides the implementations that the
loop drives. In a rebuild the edge is broken at the interface and there is no cycle.

Two subdirectories are chapters of their own:

- [`tri-colliderknoopc`](tri-colliderknoopc/README.md) — the custom triangle-mesh collider.
- [`dcylinder`](dcylinder/README.md) — the cylinder primitive the dynamics library lacks.

## The load-bearing ideas

Named once here so the twins can be terse.

**The fixed timestep is the clock.** The world accumulates real elapsed time and runs whole
steps of a fixed size, never a partial one. Everything physical is expressed in steps, not
seconds: "apply this force for two hundred steps", "this object was created at step *n*".
That is what makes the simulation reproducible, and reproducibility is
[criterion 8](../../SYSTEM-REQUIREMENTS.md#6-conformance). The accumulator is bounded, so a
long frame produces a bounded catch-up burst rather than an unbounded one. Rendering between
steps is handled by interpolating each body's last two solved placements.

**A step has four phases and their order is fixed.** Collision detection, *tune*, solve,
*data update*. The two engine-owned phases are the reason the seam demands a solver that
separates detection from solving: tune runs after contacts exist and before the solve, and
is where the engine rewrites contact parameters, injects its own constraints and applies
effectors; data update runs after the solve and is where results are read back, transforms
are published and sleep is decided. Almost every callback in this chapter belongs to one of
those two phases, and putting one in the wrong phase is the most common way to break
determinism.

**Shell, element, joint.** A physical object is a *shell*. A shell owns *elements* — one
rigid body each, with a set of collision shapes and a bone it is bound to — and *joints*
between them. A shell is built from the skeleton's own authored data: which bones are
physical, what shape each has, what joint type connects it to its parent, and what limits
that joint has. One game object, one shell; one bone, at most one element.

**Ownership of a bone's transform is exclusive and it changes at run time.** At any moment a
bone is driven either by the animation system or by its element, never both. A ragdoll is
the moment every bone changes hands; a partly-physical object (a swinging door, a corpse
still holding a weapon) has the boundary in the middle of the skeleton. The handover is the
single most error-prone thing in the chapter, and the twins say at each site which side owns
what. The blend back from physics to animation, and the settling rule that decides when a
ragdoll has come to rest and may stop being simulated, are stated in the shell twins.

**A character is not a rigid body.** It is one body with its orientation overwritten to
upright every step, driven by an *intent* — a direction it wants to go and a speed cap —
rather than by forces. Ground detection, step climbing, slope limits, crouch and jump are
all decisions made outside the solver and imposed on it. See
[`PHCharacter.h`](PHCharacter.h.md) first, then
[`PHSimpleCharacter.h`](PHSimpleCharacter.h.md) for the machinery and the two concrete
controllers for the player and for creatures.

**Islands.** Every object owns a private solver world. When two objects touch, their worlds
are spliced together for one step and split apart again afterwards, so the solver only ever
sees one connected system at a time and unrelated objects never share a matrix. Sleep is
decided per island, not per body.

**Sleeping is aggressive and its thresholds are per-class.** A world of several hundred
physical props cannot be solved every step, so bodies that have settled stop being
simulated. The detector uses two windows — a short one and a long one — over velocity and
acceleration, and the thresholds differ by an order of magnitude or more between a crate, a
ragdoll and a character. Getting them wrong is not a performance bug; a character that
sleeps stops responding to the ground moving under it.

**Collision filtering happens before any contact is computed**, in two independent
mechanisms: *groups* (these specific objects are parts of one thing) and *classes* (this kind
of thing ignores that kind). See [`PHCollideValidator.h`](PHCollideValidator.h.md).

**Activation: find the place before you create the thing.** Whenever something is about to
appear with a volume — a ragdoll, a spawned creature, an object switching from animated to
simulated, a blast radius — a temporary body is settled at the requested place with the rest
of the world frozen, and where it comes to rest is where the real thing goes. A settle that
does not converge is a refusal, not a warning. This is what keeps
[criterion 9](../../SYSTEM-REQUIREMENTS.md#6-conformance) — ragdolls and vehicles without
explosion or tunnelling — reachable at all.

**Breakage is a stress calculation the solver never sees.** A breakable object carries
*fractures* (seams inside one rigid body) and *breakable joints*. Each step, the force the
solver is spending to hold the seam or the joint together is read back and compared against
an authored threshold; when it is exceeded, the body is split or the joint destroyed, and
the blow that caused it is replayed onto the fragment. Impacts are recorded for that replay.

**Everything defends against non-finite numbers.** Safe-value wrappers keep a last known-good
copy of a body's state and substitute it when the solver returns nonsense; positions are
tested against the level's bounds; network state is validated on import. Treat these as
load-bearing, not as debug scaffolding.

**The script surface is frozen.** The physics types, methods and callback names exported to
Lua are used by the shipped scripts and cannot be renamed.

## What is *not* here

**Vehicles.** A car is assembled in [`src/xrGame`](../xrGame/README.md) — its engine curve,
its gearbox, its steering and its damage model are game logic. What this chapter supplies is
the substrate they are built from: a chassis element, one element per wheel, and the
two-axis joint with stops and motors that is a suspension strut and a steer axis at once
(see [`PHJoint.h`](PHJoint.h.md)). A rebuilder reading this chapter for "how do vehicles
work" will find the joint semantics and nothing above them, and that split is correct: the
seam's own warning applies here, since the shipped vehicle tuning values are expressed in
the original dynamics library's units and transfer least cleanly of anything in the game
data.

**The constraint solve itself**, which is the seam, and **the collision tree**, which is
[`src/xrCDB`](../xrCDB/README.md).

## Twins

Alphabetical within each of the module's concerns; the table covers every file in the
directory, including those written by other hands.

### The world and the step

| File | Role |
|---|---|
| [`IPHWorld.h`](IPHWorld.h.md) | The port the engine drives physics through: gravity, the fixed step, the deferred-call hook. |
| [`PHWorld.h`](PHWorld.h.md) · [`PHWorld.cpp`](PHWorld.cpp.md) | The fixed-timestep accumulator, the four object registries, and the strict phase order inside one step. |
| [`PHWorldScript.cpp`](PHWorldScript.cpp.md) | The world as Lua sees it: gravity, time factor, deferred calls. |
| [`PHObject.h`](PHObject.h.md) · [`PHObject.cpp`](PHObject.cpp.md) | The base every simulated unit derives from: activation, freezing, deferred disabling, and the broadphase query that turns neighbours into contacts. |
| [`PHUpdateObject.h`](PHUpdateObject.h.md) | The lighter participant: something that wants the pre- and post-solve callbacks without being collidable. |
| [`PHIsland.h`](PHIsland.h.md) · [`PHIsland.cpp`](PHIsland.cpp.md) | Private solver worlds spliced together for one step and split apart again; solver choice and non-finite scrubbing. |
| [`PHItemList.h`](PHItemList.h.md) | An intrusive list, so an object can leave the world's active list in constant time without knowing its position. |
| [`PHInterpolation.h`](PHInterpolation.h.md) · [`PHInterpolation.cpp`](PHInterpolation.cpp.md) | The two-sample ring that lets the renderer draw between physics steps. |
| [`PHDisabling.h`](PHDisabling.h.md) · [`PHDisabling.cpp`](PHDisabling.cpp.md) | The two-window sleep detector and its three mixes: translational, rotational and both. |
| [`DisablingParams.h`](DisablingParams.h.md) · [`DisablingParams.cpp`](DisablingParams.cpp.md) | The default sleep thresholds and the per-object overrides read from configuration. |
| [`PHCommander.h`](PHCommander.h.md) · [`PHCommander.cpp`](PHCommander.cpp.md) | The condition/action list evaluated once per step — "when this becomes true, do that". |
| [`PHReqComparer.h`](PHReqComparer.h.md) | How two standing requests are recognized as the same one, so installing twice does not stack. |
| [`PHSimpleCalls.h`](PHSimpleCalls.h.md) · [`PHSimpleCalls.cpp`](PHSimpleCalls.cpp.md) | The two ready-made standing requests: a step timer and a constant thruster. |
| [`PHSimpleCallsScript.cpp`](PHSimpleCallsScript.cpp.md) | The frozen names by which a script builds those two. |
| [`PHScriptCall.h`](PHScriptCall.h.md) · [`PHScriptCall.cpp`](PHScriptCall.cpp.md) | The eight shapes of script-supplied condition and action, and the one-shot retirement rule. |

### Shells, elements and joints

| File | Role |
|---|---|
| [`PhysicsShell.h`](PhysicsShell.h.md) · [`PhysicsShell.cpp`](PhysicsShell.cpp.md) | The four abstract types every physical object is built from, and the construction of a shell from a game object's skeleton. |
| [`PHShell.h`](PHShell.h.md) · [`PHShell.cpp`](PHShell.cpp.md) | A skeleton turned into a body-and-joint graph, and the answers to the game layer's two hard questions: where is it, and who owns its bones. |
| [`PHShellActivate.cpp`](PHShellActivate.cpp.md) | Switching a shell between animated and simulated control without a visible snap. |
| [`PHShellBuildJoint.h`](PHShellBuildJoint.h.md) | Turning an authored bone-limit description into a joint of the right kind with the right stops. |
| [`PHShellNetState.cpp`](PHShellNetState.cpp.md) | A multi-body object's network state is its elements' states, in element order. |
| [`PHElement.h`](PHElement.h.md) · [`PHElement.cpp`](PHElement.cpp.md) | One rigid body: mass assembled from authored shapes, per-step velocity bleed and clamp, and the bone binding. |
| [`PHElementInline.h`](PHElementInline.h.md) | The frame conversions an element performs constantly: body origin (the centre of mass) against the element's visible frame, and element against bone. |
| [`PHElementNetState.cpp`](PHElementNetState.cpp.md) | One body's complete state in and out of a packet, including the previous interpolation sample. |
| [`PHJoint.h`](PHJoint.h.md) · [`PHJoint.cpp`](PHJoint.cpp.md) | Five constraint shapes behind one axis-oriented surface, and the axis-and-angle extraction the limits are built on. |
| [`PHJointDestroyInfo.h`](PHJointDestroyInfo.h.md) · [`PHJointDestroyInfo.cpp`](PHJointDestroyInfo.cpp.md) | Whether the force holding a joint together has exceeded what it can bear. |
| [`PHFracture.h`](PHFracture.h.md) · [`PHFracture.cpp`](PHFracture.cpp.md) | A breakable seam inside one body: the stress across it, and the split that keeps every other seam's bookkeeping correct. |
| [`PHShellSplitter.h`](PHShellSplitter.h.md) · [`PHShellSplitter.cpp`](PHShellSplitter.cpp.md) | The per-step driver that runs every fracture and breakable joint on a shell and performs the separations they ask for. |
| [`PHSplitedShell.h`](PHSplitedShell.h.md) · [`PHSplitedShell.cpp`](PHSplitedShell.cpp.md) | The cheap shell a debris fragment gets: static-only collision, a capped spatial footprint, and exit from the simulation when it stops. |
| [`PHImpact.h`](PHImpact.h.md) | One recorded blow — force, point, shape — kept so it can be replayed onto the fragment it separated. |
| [`ShellHit.cpp`](ShellHit.cpp.md) | How a hit becomes motion: a bullet is one impulse at a point, an explosion is a scattered set. |
| [`PHStaticGeomShell.h`](PHStaticGeomShell.h.md) · [`PHStaticGeomShell.cpp`](PHStaticGeomShell.cpp.md) | A body-less collision presence for an object that never moves but must still be hit. |
| [`IPHStaticGeomShell.h`](IPHStaticGeomShell.h.md) | The port for building one of those. |
| [`PhysicsShellAnimator.h`](PhysicsShellAnimator.h.md) · [`PhysicsShellAnimator.cpp`](PhysicsShellAnimator.cpp.md) | Dragging a shell's bodies toward an animation's pose by welding each to a moving target. |
| [`PhysicsShellAnimatorBoneData.h`](PhysicsShellAnimatorBoneData.h.md) | One controlled bone: the body being dragged and the constraint doing the dragging. |
| [`PhysicsShellScript.cpp`](PhysicsShellScript.cpp.md) | The slice of the physics object model Lua may touch, and the names it touches it by. |
| [`PHGeometryOwner.h`](PHGeometryOwner.h.md) · [`PHGeometryOwner.cpp`](PHGeometryOwner.cpp.md) | A set of collision shapes owned as one composite: grouping, shared material and callbacks, derived mass properties. |
| [`PHDefs.h`](PHDefs.h.md) | The collection type names the shell machinery passes around. |

### Characters

| File | Role |
|---|---|
| [`PHCharacter.h`](PHCharacter.h.md) · [`PHCharacter.cpp`](PHCharacter.cpp.md) | The contract every character controller satisfies: a body told where it wants to go, held upright, with its own sleep and network rules. |
| [`PHSimpleCharacter.h`](PHSimpleCharacter.h.md) · [`PHSimpleCharacter.cpp`](PHSimpleCharacter.cpp.md) | The shared machinery: capsule model, ground detection, step climbing, slope limits, crouch, jump, and the contact rules that make all of it work. |
| [`PHSimpleCharacterInline.h`](PHSimpleCharacterInline.h.md) | The small queries that machinery answers constantly — environment, ground normal, foot centre. |
| [`PHActorCharacter.h`](PHActorCharacter.h.md) · [`PHActorCharacter.cpp`](PHActorCharacter.cpp.md) | The player's controller: jump legality, the unclimbable material, spacing volumes, and two different collision rule sets for single-player and multiplayer. |
| [`PHActorCharacterInline.h`](PHActorCharacterInline.h.md) | Empty. |
| [`PHAICharacter.h`](PHAICharacter.h.md) · [`PHAICharacter.cpp`](PHAICharacter.cpp.md) | The creature controller: a character that can be *asked to be somewhere* and answer whether it could get there. |
| [`MovementBoxDynamicActivate.h`](MovementBoxDynamicActivate.h.md) · [`MovementBoxDynamicActivate.cpp`](MovementBoxDynamicActivate.cpp.md) | Changing a character's collision box — standing up, crouching, spawning — by growing it against a frozen world. |
| [`ElevatorState.h`](ElevatorState.h.md) · [`ElevatorState.cpp`](ElevatorState.cpp.md) | The ladder state machine: when a character is on one, how it is held there, and how it gets off. |
| [`IElevatorState.h`](IElevatorState.h.md) | The seven ladder states, published for the animation layer. |
| [`IClimableObject.h`](IClimableObject.h.md) | What a ladder must be able to answer about a character standing near it. |
| [`PHCapture.h`](PHCapture.h.md) · [`PHCapture.cpp`](PHCapture.cpp.md) · [`PHCaptureInit.cpp`](PHCaptureInit.cpp.md) | A creature's grip on a physical object: pull it in, hold it against an animated bone, let go. |
| [`IPHCapture.h`](IPHCapture.h.md) | The published four-call handle on that grip. |
| [`ActorCameraCollision.h`](ActorCameraCollision.h.md) · [`ActorCameraCollision.cpp`](ActorCameraCollision.cpp.md) | Keeping the first-person camera's near plane out of walls, by settling a tiny physical body between world steps. |

### Contacts, collision and materials

| File | Role |
|---|---|
| [`Physics.h`](Physics.h.md) · [`Physics.cpp`](Physics.cpp.md) | Contact generation: a colliding pair becomes constraints whose friction, bounce and softness come from the two surfaces' materials, with both sides free to veto or reshape each one. |
| [`PhysicsCommon.h`](PhysicsCommon.h.md) | The conversion between authored stiffness and the solver's soft-constraint parameters, plus the module's global tuning constants. |
| [`PhysicsExternalCommon.h`](PhysicsExternalCommon.h.md) · [`PhysicsExternalCommon.cpp`](PhysicsExternalCommon.cpp.md) | The vocabulary shared with the game layer, and the extraction of impact strength and struck surface from a contact. |
| [`PHCollideValidator.h`](PHCollideValidator.h.md) · [`PHCollideValidator.cpp`](PHCollideValidator.cpp.md) | Whether two objects may touch at all: groups and classes, decided before any contact is computed. |
| [`ICollideValidator.h`](ICollideValidator.h.md) | Minting a collision group, without exposing the rules. |
| [`GeometryBits.h`](GeometryBits.h.md) · [`GeometryBits.cpp`](GeometryBits.cpp.md) | The coarse per-shape static-versus-dynamic filter, applied earlier still. |
| [`Geometry.h`](Geometry.h.md) · [`Geometry.cpp`](Geometry.cpp.md) | The primitive collision shapes: building, placing, massing and querying them behind one wrapper. |
| [`ExtendedGeom.h`](ExtendedGeom.h.md) · [`ExtendedGeom.cpp`](ExtendedGeom.cpp.md) | The payload stapled onto every shape — its object, its material, its callbacks, and the triangle cache and motion history the level collider needs. |
| [`CalculateTriangle.h`](CalculateTriangle.h.md) | A triangle from the level's soup reduced to the plane, sides and distances the collider works in. |
| [`PHMoveStorage.h`](PHMoveStorage.h.md) · [`PHMoveStorage.cpp`](PHMoveStorage.cpp.md) | The shapes whose motion between steps is swept, so a fast small object cannot pass through a thin obstacle. |
| [`dRayMotions.h`](dRayMotions.h.md) · [`dRayMotions.cpp`](dRayMotions.cpp.md) | The swept-motion probe: a ray that reports its hits as if it were the body it precedes. |
| [`PHContactBodyEffector.h`](PHContactBodyEffector.h.md) · [`PHContactBodyEffector.cpp`](PHContactBodyEffector.cpp.md) | The drag a body feels from the medium it touches — water, mud, snow — as a velocity-proportional force in the contact plane. |
| [`PHBaseBodyEffector.h`](PHBaseBodyEffector.h.md) | The one thing every per-body effector needs: which body. |
| [`IColisiondamageInfo.h`](IColisiondamageInfo.h.md) | The description of an impact physics hands the game so the game can turn it into damage. |
| [`icollisiondamagereceiver.h`](icollisiondamagereceiver.h.md) | The port through which anything that can be hurt by being hit receives that description. |
| [`collisiondamagereceiver.cpp`](collisiondamagereceiver.cpp.md) | How much a collision hurts: the impact energy, split between the two surfaces. |
| [`DamageSource.h`](DamageSource.h.md) | Letting an object carry the identity of whoever set it in motion, so a thrown crate's kill is attributed. |
| [`params.h`](params.h.md) · [`params.cpp`](params.cpp.md) | The one tuning value that comes from configuration rather than from a console variable. |

### Activation and validity

| File | Role |
|---|---|
| [`IActivationShape.h`](IActivationShape.h.md) · [`IActivationShape.cpp`](IActivationShape.cpp.md) | The three ways the game asks "find me a spot for this volume that is not inside a wall". |
| [`PHActivationShape.h`](PHActivationShape.h.md) · [`PHActivationShape.cpp`](PHActivationShape.cpp.md) | The settle procedure those three compose: a temporary body grown against a frozen world. |
| [`PHValideValues.h`](PHValideValues.h.md) | The safe-value wrappers: a last known-good scalar, vector, orientation or body state, substituted whenever the solver returns something that is not a number. |
| [`ph_valid_ode.h`](ph_valid_ode.h.md) | Whether a body's state is still made of finite numbers. |
| [`phvalide.h`](phvalide.h.md) · [`phvalide.cpp`](phvalide.cpp.md) | Whether a position is inside the level, and the diagnostic when it is not. |

### Support

| File | Role |
|---|---|
| [`IPhysicsShellHolder.h`](IPhysicsShellHolder.h.md) | Everything physics is allowed to know about the game object that owns a shell — the module's whole view of the game. |
| [`MathUtils.h`](MathUtils.h.md) · [`MathUtils.cpp`](MathUtils.cpp.md) | The vector, angle and ballistics helpers the module needs. |
| [`MathUtilsOde.h`](MathUtilsOde.h.md) | The numerically careful normalize, the velocity clamp, and the restitution conversion. |
| [`matrix_utils.h`](matrix_utils.h.md) | Comparing, clamping and differentiating rigid transforms. |
| [`SpaceUtils.h`](SpaceUtils.h.md) | A collision space's bounds turned into what the engine's spatial index wants. |
| [`PHDynamicData.h`](PHDynamicData.h.md) · [`PHDynamicData.cpp`](PHDynamicData.cpp.md) | The conversion between the engine's transform convention and the solver's, and an abandoned pose cache. |
| [`CycleConstStorage.h`](CycleConstStorage.h.md) | A fixed-length ring of samples, indexed from the newest backwards. |
| [`BlockAllocator.h`](BlockAllocator.h.md) | A bump allocator reclaimed whole, for the per-step scratch the solver phases produce. |
| [`console_vars.h`](console_vars.h.md) · [`console_vars.cpp`](console_vars.cpp.md) | The module's run-time tunables and what each one buys. |
| [`debug_output.h`](debug_output.h.md) · [`debug_output.cpp`](debug_output.cpp.md) | The port physics draws and counts through when someone is watching, and its do-nothing filling. |
| [`ode_include.h`](ode_include.h.md) · [`ode_redefine.h`](ode_redefine.h.md) | The single point at which the dynamics library is pulled in, and the substitution of the engine's own scalar math inside it. |
| [`xrPhysics.h`](xrPhysics.h.md) · [`xrPhysics.cpp`](xrPhysics.cpp.md) | Which symbols the rest of the engine sees, and the routing of the dynamics library's allocations through the engine's allocator. |
| [`StdAfx.h`](StdAfx.h.md) · [`stdafx.cpp`](stdafx.cpp.md) | The set of engine modules this one compiles against. |
