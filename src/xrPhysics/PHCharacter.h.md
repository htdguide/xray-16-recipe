# src/xrPhysics/PHCharacter.h

> The contract every character controller satisfies — a body that is told where it
> wants to go rather than what forces act on it.

**Needs** — [`PHCharacter.cpp`](PHCharacter.cpp.md) · [`PHObject.h`](PHObject.h.md) · [`PHDisabling.h`](PHDisabling.h.md) · [`PHInterpolation.h`](PHInterpolation.h.md) · [`xrServerEntities/PHSynchronize.h`](../xrServerEntities/PHSynchronize.h.md) · [`xrEngine/IPhysicsShell.h`](../xrEngine/IPhysicsShell.h.md) · [`IPhysicsShellHolder.h`](IPhysicsShellHolder.h.md) · [Seam: Rigid-body dynamics](../../SYSTEM-REQUIREMENTS.md#seam-rigid-body-dynamics)
**Used by** — [`ClimableObject.cpp`](../xrGame/ClimableObject.cpp.md) · [`PHMovementControl.cpp`](../xrGame/PHMovementControl.cpp.md) · [`PHMovementDynamicActivate.cpp`](../xrGame/PHMovementDynamicActivate.cpp.md) · [`ElevatorState.cpp`](ElevatorState.cpp.md) · [`ElevatorState.h`](ElevatorState.h.md) · [`IClimableObject.h`](IClimableObject.h.md) · [`IPHCapture.h`](IPHCapture.h.md) · [`MovementBoxDynamicActivate.cpp`](MovementBoxDynamicActivate.cpp.md) · [`MovementBoxDynamicActivate.h`](MovementBoxDynamicActivate.h.md) · [`PHActorCharacter.h`](PHActorCharacter.h.md) · [`PHCapture.cpp`](PHCapture.cpp.md) · [`PHCapture.h`](PHCapture.h.md) · [`PHCaptureInit.cpp`](PHCaptureInit.cpp.md) · [`PHCharacter.cpp`](PHCharacter.cpp.md) · _and 3 more_
**Tier floor** — T1: it owns a solver body directly and exposes its state for network
serialization.

## Purpose

**An actor is not an ordinary rigid body**, and this is where that is stated. A rigid body
is driven by forces and rotates freely; a character is driven by an *intent* — a direction
it wants to move, a speed cap, a jump — and must stay upright no matter what hits it. This
header is the full surface of that difference, and it is a substantive interface rather than
a declaration, because it is what any concrete controller must satisfy.

The class also carries four roles at once, and separating them is the first thing a
rebuilder should do: it is a simulated object in the world, a network-synchronizable state,
a sleep-managed body, and the engine's abstract physics element. Those are four independent
concerns fused by inheritance because C++ made that cheap.

## State

```text
RECORD Character
  body              : Body          # exactly ONE body; a character is never multi-body
  exists            : bool          # false before Create and after Destroy
  creation_step     : int (64-bit)  # the world step it was created on
  interpolation     : Interpolation # previous positions, for render-time smoothing
  holder            : ShellHolder   # the game object
  mass              : real
  was_enabled_before_freeze : bool

  # --- the surface underfoot ---
  last_material     : reference to int (16-bit)   # indirect: see Notes
  injurious_material: int (16-bit)   # a material that damages on contact, if any

  # --- safe state, for recovery from a non-finite solver result ---
  safe_velocity     : vector
  safe_position     : vector
  mean_y            : real

  # --- mutual-exclusion volumes ---
  restriction_type      : ERestrictionType   # which size class this character is
  new_restriction_type  : ERestrictionType   # pending change, applied between steps
  object_radius         : real               # the character's own radius, for size choice
  in_touch_restrictor   : bool
  actor_movable         : bool               # may the player push this character?
```

**Invariants** — `exists` gates every operation: a character whose body has not been created
answers every query with a neutral value rather than failing. The restriction type changes
only between steps, never inside a contact callback, which is why there are two fields.

## the intent surface

**Contract** — this is the part a rebuild must reproduce exactly, because the game layer
speaks to characters only through it.

- **Desired motion** — set an acceleration (the direction and strength the character *wants*
  to move), a maximum velocity, an air-control factor, and a camera direction. The camera
  direction is not for rendering: it feeds the climbing state machine and the jump.
- **Jump** — request a jump with a velocity vector; ask whether a jump is in progress. Each
  concrete controller decides whether a jump is currently legal.
- **Environment** — ask whether the character is on the ground, against a wall, or in the
  air; ask for the ground normal; ask whether a contact happened since the last ask (a
  one-shot latch, used to trigger footstep and impact sounds).
- **Placement** — set and get the position, get the body position (which differs: see
  below), get the foot centre, get the interpolated position.
- **Forces** — apply a force or an impulse, add a control velocity. These bypass the intent
  and act on the body directly; they are the mechanism for explosions and for being pushed.
- **Material** — set the character's own material, read the last material it stood on, read
  an injurious material it is standing in.
- **Restrictors** — the size class and the radius, and a hook for a character to be told it
  has touched another's restrictor volume.

**Notes** — the *last material* is held as an indirect reference rather than a value, and
the reason is the vehicle and mounted cases: when a character is riding something, the
surface underfoot is the vehicle's, and redirecting the reference is how that is expressed
without the controller knowing what a vehicle is. A rebuild can do the same with an optional
override field, which is clearer.

`GetPosition` and `GetBodyPosition` are different: the body's origin is not the character's
reported position, because the collision shape is a capsule whose centre sits at mid-body
while the game wants the point on the ground. Confusing them puts every character half their
height into the floor.

## the upright constraint

**Contract** — a character's body has its angular velocity zeroed and its orientation reset
to identity, every step. Exposed as `fix_body_rotation`.

**Invariants** — a character's reported angular velocity is always zero, and its network
state always carries the identity orientation. A rebuild that lets the capsule tumble has
built a ragdoll, not a character.

**Notes** — the facing direction of a character is *not* in its physics state at all. It
lives in the game object's transform and is driven by animation and by look input. Physics
owns only the position.

## `CutVelocity`

**Contract** — clamp the body's linear velocity to a limit, and — this is the part that
matters — *advance the body by one step under the removed velocity first*, so that the
clamp changes the speed without changing where the body would have been. Angular velocity is
zeroed outright.

**Notes** — this is used by the activation procedures, which need to move a body a bounded
distance per iteration without introducing position error. Naively clamping velocity
teleports the body backwards relative to where the solver just put it.

## network state

**Contract** — export and import position, previous position, linear velocity, standing
force, and the enabled flag. Angular velocity, orientation and torque are exported as
identity/zero and ignored on import, because of the upright constraint.

**Invariants** — the imported state must leave the body in a valid configuration; this is
asserted. A character whose network state arrives with a non-finite position is the classic
way a client's whole physics world dies.

**Notes** — the *previous* position travels alongside the current one so the receiving end
can rebuild the interpolation window and render smoothly without waiting a second packet.

## the three shapes of the controller

**Contract** — `CastActorCharacter` and `CastAICharacter` let a caller discover which
concrete controller it holds. There are two:
[`PHActorCharacter.h`](PHActorCharacter.h.md) (the player) and
[`PHAICharacter.h`](PHAICharacter.h.md) (everything else), both on a shared base
([`PHSimpleCharacter.h`](PHSimpleCharacter.h.md)).

## `create_ai_character` / `create_actor_character`

**Contract** — the module's two character factories, exported so the game layer can make one
without naming the concrete class. The actor factory takes a flag distinguishing
single-player from multiplayer, because the two want different collision rules between
players — see [`PHActorCharacter.cpp`](PHActorCharacter.cpp.md).

## `virtual_move_collide_callback`

**Contract** — a contact callback for probing a character's movement without committing to
it: it creates a frictionless, slightly soft contact attached to only one of the two bodies,
so the probe is stopped by the world but the world is not pushed by the probe. Declared here
because the game layer installs it.

## Notes

`ERestrictionType` (declared in [`PhysicsExternalCommon.h`](PhysicsExternalCommon.h.md))
enumerates size classes — stalker, small stalker, medium monster, actor, none. They are not
about collision shape: they are about which *mutual-exclusion volume* a character projects
so that creatures of different sizes keep sensible distances from each other. See
[`PHActorCharacter.cpp`](PHActorCharacter.cpp.md).

`EEnvironment` — on the ground, at a wall, in the air — is the controller's published
summary of its contact situation, and is what the animation layer branches on.
