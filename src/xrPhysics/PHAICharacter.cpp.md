# src/xrPhysics/PHAICharacter.cpp

> Lets a creature's brain ask "can I be here next frame?" and have the collision
> world answer honestly, by walking the request through the solver in small steps and
> refusing if any of them is blocked.

**Needs** — [`PHAICharacter.h`](PHAICharacter.h.md) · [`PHSimpleCharacter.h`](PHSimpleCharacter.h.md) · [`ExtendedGeom.h`](ExtendedGeom.h.md) · [`Physics.h`](Physics.h.md) · [Seam: Rigid-body dynamics](../../SYSTEM-REQUIREMENTS.md#seam-rigid-body-dynamics)
**Used by** — [`PHAICharacter.h`](PHAICharacter.h.md)
**Tier floor** — T1: it drives individual solver steps by hand and manipulates the body's
gravity mode mid-procedure.

## Purpose

Creature movement is decided by the navigation layer ([chapter 14](../xrAICore/README.md)),
which produces a position for the next frame from a path, not from forces. But a creature
that simply teleports to that position walks through doors, through other creatures and
through each other. This file is the reconciliation: **the AI proposes, the collision world
disposes.**

It is the clearest statement in the chapter of the desired-versus-permitted separation, and
it is worth understanding before reading the actor controller, which solves the same problem
the other way round.

## `TryPosition`

**Contract** — given a target position, attempt to move the creature there through the
collision world. Returns true and leaves the creature at (or very near) the target when the
whole path was clear; returns false and leaves the creature wherever it got stuck, with its
original velocity restored.

```text
FUNCTION try_position(target, exact) -> bool
  IF the body does not exist                      RETURN false
  IF forced physics control OR mid-jump           RETURN false   # not ours to decide
  IF a collision is already being resolved        RETURN false

  current = position
  saved_velocity = velocity
  displacement = target - current
  IF displacement is zero OR the frame took no time    RETURN true

  # Choose a step size of one FOOT RADIUS, so no step can pass through a wall.
  step_length = foot_radius
  whole_steps = floor(|displacement| / step_length)
  IF whole_steps > 15                             # cap the work per frame
    whole_steps = 15
    step_length = |displacement| / 15
  remainder = |displacement| - whole_steps * step_length

  saved_gravity_mode = body gravity mode
  turn gravity OFF                                # the AI's path is already ground-aware

  FOR i IN 1..whole_steps
    set velocity so one fixed step covers step_length along the displacement
    wake the body
    IF NOT step_single()                          # the step was blocked
      restore saved_velocity
      succeeded = false
      BREAK
  IF not blocked
    set velocity to cover the remainder in one step
    wake the body
    succeeded = step_single()

  restore saved_gravity_mode
  restore saved_velocity
  reached = position                              # wherever the walk actually ended
  set position to `reached`
  last_move = (reached - current) / frame_duration      # for the animation layer
  refresh the interpolation window TWICE                # see Notes
  IF succeeded   put the body back to sleep
  clear any recorded collision damage
  RETURN succeeded
```

**Invariants** — gravity is off for the whole procedure and restored afterwards, whatever
happens. The creature's velocity is restored on both paths: the walk is a *query*, and must
not leave the body carrying the query's velocity into the next real step.

**Notes** — several decisions here are easy to get wrong in a rebuild.

**The step length is the foot radius.** That is the tunnelling guarantee: a step can never
carry the capsule more than its own radius, so it cannot pass entirely through a thin wall
between two steps. Choosing a fixed step length in metres instead would tunnel for small
creatures and waste steps for large ones.

**The cap of fifteen steps** bounds the per-frame cost. A creature asked to move further
than fifteen foot-radii in one frame gets *longer* steps instead of more of them, trading
the tunnelling guarantee for a bounded budget — which is the right trade, because such a
request means the creature is already moving unrealistically fast.

**Gravity is off** because the navigation layer's position is already on the navigation
mesh, i.e. already on the ground. Leaving gravity on would make each probe step also fall,
and the creature would sink into the floor a little on every frame.

**The interpolation window is refreshed twice.** That collapses the render-time smoothing
window onto the new position, so the creature does not visibly slide from where it was. Once
would leave one stale sample.

**The body is put back to sleep on success.** This is the inversion stated in the header: a
creature that is successfully following its path is not simulated at all between frames. Only
a creature whose path was blocked stays awake, so that the solver can sort out the collision
it ran into.

`last_move` is divided by the *frame* duration, not the step duration, because it is consumed
by the animation layer, which runs per frame.

The clearing of collision damage at the end prevents the probe's own contacts from being
read as a real impact by the game.

## `InitContact`

**Contract** — the creature's contact rules, applied on top of the shared ones.

```text
FUNCTION init_contact(contact, do_collide, mat_a, mat_b)
  IF either material is flagged an actor obstacle
    force do_collide TRUE                # this material exists to stop creatures
  run the shared character contact rules
  IF the creature is under control, has lost control, or is jumping
    contact.friction = 0                 # let it slide rather than catch
  IF BOTH sides of the contact are characters
    mark "standing on an object" and invalidate the wall-contact record
```

**Notes** — the actor-obstacle material flag is a level-authoring tool: an invisible surface
that creatures collide with and nothing else does, used to keep them out of places the
geometry would otherwise allow. It forces collision *on* rather than off, which makes it the
only material flag in the chapter that adds contacts instead of removing them.

Zeroing friction while under control is what stops a creature from catching on a corner
while the AI is walking it past: with friction, the probe steps above would report the path
blocked for a wall the creature could have slid along.

The character-on-character case sets the on-object flag, which downstream lets a creature
stand on another creature's head without the wall-contact logic deciding it is pinned. It
also disables the wall-contact record, because another character is not a wall and should not
stop a climb.

## `Jump`

**Contract** — unconditionally marks a jump pending with the given velocity.

**Notes** — the contrast with the player's jump is the point: the player's controller tests
ground normal, slope material and lost-control state before allowing a jump, because the
player may press the key at any moment. A creature's brain has already established it can
jump, and second-guessing it here would make creatures refuse jumps the navigation layer
counted on.

## `Create` and `ValidateWalkOn`

**Contract** — creation delegates and clears the forced-control flag; walk-on validation
delegates unchanged.

**Notes** — the redundant clear of the forced-control flag in `Create` (it is already cleared
in the constructor) guards the reuse case: a creature object is recreated on respawn without
being reconstructed.

## static damage

**Contract** — overridden to do nothing. Creatures take no collision damage from static
level geometry.

**Notes** — this is a game decision, not a physics one, and it is deliberate: creatures are
constantly scraped along walls by the probe walk above, and any static-damage rule would
kill them slowly for no reason the player can see. Player characters keep it, which is why
falling hurts the player and not a dog.
