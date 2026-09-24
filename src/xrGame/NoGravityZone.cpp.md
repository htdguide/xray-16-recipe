# src/xrGame/NoGravityZone.cpp

> An anomaly that switches gravity off for everything inside it, and gives each object one upward nudge on the way in so that it actually leaves the ground.

**Needs** — [`NoGravityZone.h`](NoGravityZone.h.md) · [`CustomZone.h`](CustomZone.h.md) · [`PHMovementControl.h`](PHMovementControl.h.md) · [`CharacterPhysicsSupport.h`](CharacterPhysicsSupport.h.md) · [`entity_alive.h`](entity_alive.h.md) · [`xrPhysics/PhysicsShell.h`](../xrPhysics/PhysicsShell.h.md) · [`xrPhysics/IPHWorld.h`](../xrPhysics/IPHWorld.h.md) · [Seam: Rigid-body dynamics](../../SYSTEM-REQUIREMENTS.md#seam-rigid-body-dynamics)
**Used by** — [`NoGravityZone.h`](NoGravityZone.h.md)
**Tier floor** — T3: two flag flips and one impulse per object

## Purpose

Turning gravity off is not enough to make something float. An object resting on the ground is
in contact, and removing gravity leaves it resting: the contact constraint holds it there and
nothing ever lifts it. This zone therefore does two things — clears the gravity flag, and
delivers one impulse equal to exactly one timestep's worth of weight, which is the smallest
push that breaks the contact.

The second thing this file shows is that a *creature* and an *object* need entirely different
handling. An object has a rigid-body shell; a creature has a character controller that
synthesizes its own gravity and must additionally be told to hand control to the physics.

## State

`Stateless.` Everything it changes lives on the objects it touches.

## `enter_Zone` · `exit_Zone`

**Contract** — switch gravity off as an object enters the zone and back on as it leaves. Both
run the base zone's own transition, and the order differs between them: entry runs the base
first, exit runs the gravity restore first.

**Invariants** — gravity must be restored *before* the base exit runs, because the base exit
may drop the object from the zone's tracked set, after which the zone no longer knows to
restore it. An object that leaves the zone during the same tick it is removed would otherwise
float forever.

## `UpdateWorkload`

**Contract** — the per-tick pass over the zone's contents, re-asserting "no gravity" on every
object inside.

**Notes** — re-asserting every tick rather than only on entry is defensive: anything else that
touches an object's gravity flag — a ragdoll activation, a script, another zone — would
otherwise win permanently. It also means the flag is written far more often than it changes,
which is cheap enough not to matter.

## `switchGravity`

**Contract** — turns gravity on or off for one object, by whichever of the two mechanisms
applies. Skips objects that are being destroyed. Does nothing for an object that is neither a
physics body nor a living creature.

```text
FUNCTION switchGravity(entry, gravity_on)
  IF entry.object is being destroyed THEN RETURN
  holder = entry.object as a physics holder; IF none THEN RETURN
  shell  = holder.physics body

  IF shell exists AND is active THEN
    shell.apply_gravity = gravity_on
    IF turning gravity OFF and it was previously on THEN
      # One nudge, to break the resting contact. Pick a random element of the body and
      # a random point on it, so that the body tumbles rather than rising flat.
      element = a random element of the body
      IF element is active THEN
        element.apply_impulse_at(random point within its radius,
                                 random direction,
                                 magnitude: body mass * gravity * one fixed timestep)
      END IF
    END IF
    RETURN
  END IF

  # No rigid body: a living creature, driven by its character controller.
  IF entry.object is alive THEN
    movement = the creature's character movement controller
    movement.apply_gravity          = gravity_on
    movement.forced_physics_control = NOT gravity_on
    IF turning gravity OFF AND the creature is standing on the ground THEN
      movement.apply_impulse(along the ground normal,
                             magnitude: creature mass * gravity * one fixed timestep)
    END IF
  END IF
```

**Invariants**

- The impulse magnitude is `mass * gravity * fixed_step` — the momentum gravity would have
  added in exactly one simulation step. That is the *smallest* impulse that reliably separates
  a resting body from its contact, and choosing anything larger makes objects visibly jump as
  they cross the zone boundary. The fixed timestep appearing in a gameplay formula is unusual
  and is the reason this constant is worth writing down.
- The creature path also sets *forced physics control*, which hands the creature's motion to
  the physics solver instead of its normal walking controller. Without it the creature keeps
  walking on air; with it, it tumbles. Clearing gravity alone does not float a creature.
- The impulse is only applied when gravity is being turned *off* and only when the object was
  still under gravity (or, for a creature, still on the ground). Applying it every tick would
  accelerate everything upward indefinitely.

**Notes** — the random element and random direction for the object case are deliberate: a
single impulse through the centre of mass lifts a body without rotating it, which reads as a
lift rather than as weightlessness. Off-centre and randomly directed, it tumbles.

The creature branch does not check that the object really is a living creature before
dereferencing its controller; it relies on the zone's own classification of the entry as
"not a non-living object". A rebuild should test for the controller directly.
