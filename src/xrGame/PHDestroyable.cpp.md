# src/xrGame/PHDestroyable.cpp

> Turning one physically simulated object into several: spawning the debris, waiting for every piece to arrive, and handing each one the momentum of the blow that broke it.

**Needs** — [`PHDestroyable.h`](PHDestroyable.h.md) · [`PHDestroyableNotificate.h`](PHDestroyableNotificate.h.md) · [`PHSkeleton.h`](PHSkeleton.h.md) · [`PhysicsShellHolder.h`](PhysicsShellHolder.h.md) · [`Hit.h`](Hit.h.md) · [`Level.h`](Level.h.md) · [`Actor.h`](Actor.h.md) · [`CharacterPhysicsSupport.h`](CharacterPhysicsSupport.h.md) · [`object_factory.h`](../xrServerEntities/object_factory.h.md) · [`xrServerEntities/xrServer_Objects_ALife.h`](../xrServerEntities/xrServer_Objects_ALife.h.md) · [`xrPhysics/PhysicsShell.h`](../xrPhysics/PhysicsShell.h.md) · [`Include/xrRender/Kinematics.h`](../Include/xrRender/Kinematics.h.md) · [Seam: Rigid-body dynamics](../../SYSTEM-REQUIREMENTS.md#seam-rigid-body-dynamics)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: object lifecycle plus rigid-body state transfer

## Purpose

When a crate, a barrel or a corpse breaks apart, the engine does not tear one body into
pieces. It **spawns entirely new entities** carrying the debris visuals and then transfers
the original's motion to them, so that the pieces fly apart as if they had been part of it.

That indirection is the whole design, and it is forced: the pieces are separate spawns with
their own identities so they can be networked, saved, and aged out independently. The
consequence is that destruction is **asynchronous** — a spawn request goes out, the replies
come back over some number of frames, and the original may not vanish until every piece has
reported in.

This file is the bookkeeping for that wait, plus the momentum transfer at the end of it.

## State

```text
RECORD PHDestroyable
  destroyed_obj_visual_names : list<text>   # one visual per piece; duplicates allowed
  notificate_objects         : list<Piece>  # pieces that have arrived
  depended_objects           : int          # pieces spawned but not yet arrived
  fatal_hit                  : Hit          # the blow that broke it: bone, point, direction, impulse
  flags : { destroyable, destroyed, released }
```

**Invariants**
- `released` means "no piece is still owed to me". It starts true, goes false the moment a
  piece is spawned naming this object as its source, and returns to true when the last piece
  has reported. The original may not be removed while it is false.
- `depended_objects` counts outstanding pieces, and reaching zero is what triggers the
  momentum transfer for *all* of them at once. Pieces are not animated as they arrive —
  they are held invisible and inert until the set is complete, so that the debris appears in
  one instant rather than trickling in.
- Pieces are spawned only on the authoritative side and only in single player.

## `Destroy`

**Contract** — begins the destruction. Refuses when the object is not destroyable or is
already destroyed. Marks the object's skeleton as not worth saving, re-enables its body (a
disabled body cannot hand over velocities), registers it with the scheduler so the removal
poll runs, spawns one piece per configured visual, and marks itself destroyed.

```text
FUNCTION destroy(source_id, section = "physics skeleton object")
  IF NOT destroyable OR already destroyed
    RETURN
  clear the arrived-pieces list
  the object's skeleton is no longer worth saving
  enable the object's physics body            # its velocities are about to be read
  register with the scheduler                 # so the removal poll runs
  IF source_id IS this object's own identity
    released = false                          # we now owe ourselves pieces
  IF single player
    FOR EACH visual IN destroyed_obj_visual_names
      spawn a piece with that visual, naming source_id as its source
  destroyed = true
```

**Notes** — the released flag is cleared only when the source identity is this object's own.
A destruction triggered on behalf of some *other* object leaves this one immediately
removable. That asymmetry is what lets a chain of destructions collapse without every link
waiting on the next.

Outside single player nothing is spawned at all: the object simply becomes destroyed and
vanishes. Debris is a single-player-only effect.

## `GenSpawnReplace`

**Contract** — creates one piece: a server record of the given class, carrying one debris
visual and a back-reference to the object it came from. On the authoritative side the record
is broadcast as a spawn and then released, and the outstanding count goes up by one.

**Invariants** — the back-reference is what lets the arriving piece find its source and
report in. It is the only link between a spawned entity and the object it is a fragment of.

## `InitServerObject`

**Contract** — fills the piece's record from the original's placement: position, orientation
decomposed into angles, and both navigation identities. The cross-level graph identity is
taken from the *current level's* vertex when the alife simulation is running and left invalid
when it is not — a piece of debris is not worth placing in the world simulation, only in the
loaded level.

The record is marked as a **local** spawn with no respawn, no parent and no name.

## `spawn_notificate` (in [`PHDestroyableNotificate.cpp`](PHDestroyableNotificate.cpp.md))

The other half of the handshake. See that file.

## `NotificateDestroy`

**Contract** — one piece has arrived. Decrements the outstanding count, hides and freezes the
piece, and remembers it. When the count reaches zero — every piece is present — the momentum
transfer runs over all of them, the original is physically removed, and the object becomes
releasable.

**Invariants** — must not run while the physics world is mid-step. Everything it does mutates
bodies, and the solver holds them during a step.

```text
FUNCTION on_piece_arrived(piece)
  REQUIRE depended_objects > 0
  REQUIRE the physics world is not stepping
  depended_objects -= 1
  hide and freeze the piece          # it waits, inert, for its siblings
  remember it
  IF depended_objects == 0
    FOR EACH remembered piece
      transfer momentum into it and reveal it
    physically remove the original
    forget them all
    released = true
```

## `NotificatePart` — the momentum transfer

**Contract** — the payoff. Places one piece exactly where the original was and gives every
one of its rigid elements the original's motion at that element's centre of mass, plus a
share of the fatal blow, plus a randomized scatter impulse. This is what makes debris fly
apart convincingly rather than merely dropping.

```text
FUNCTION transfer_into(piece)
  place the piece at the original's current dynamic transform

  # five factors, defaulted, then overridden by the ORIGINAL's model data,
  # then overridden again by the PIECE's own model data. The piece wins,
  # which is how a specific fragment overrides its parent's defaults.
  random_min           = 1     # scatter impulse per unit of mass
  random_hit_impulse   = 1     # extra scatter, per unit of the fatal blow
  reference_bone       = the original's root bone
  impulse_factor       = 1     # how much of the fatal blow each element receives
  linear_vel_factor    = 1
  angular_vel_factor   = 1
  apply overrides from the original's "impulse transition to parts" section
  apply overrides from the piece's "impulse transition from source bone" section

  source = the original's rigid element for reference_bone

  FOR EACH element IN the piece
    scatter = random_min * element.mass
    IF the fatal hit is valid and named a bone
      # apply the blow at the point on the ORIGINAL where it landed,
      # transformed into world space through that bone
      point = the fatal hit's bone-space position, through that bone's
              transform, through the original's placement
      element.apply_impulse_at(point, fatal_hit.direction,
                               fatal_hit.impulse * impulse_factor)
      scatter += random_hit_impulse * fatal_hit.impulse
    element.apply_impulse(a random direction, scatter)
    # inherit the original's motion AT THIS ELEMENT'S location, not at its own
    # centre — a spinning object throws its extremities harder
    element.linear_velocity  = source.velocity_at(element.mass_centre) * linear_vel_factor
    element.angular_velocity = source.angular_velocity * angular_vel_factor

  enable the piece's body and collision; make it visible
  IF the original was in a collision group, put the piece in the same one
  set the piece's automatic removal time from whichever model data declares it
```

**Invariants** — the linear velocity is sampled *at each element's own position* on the
original body, not at the body's centre. That is the difference between debris that spreads
and debris that moves as a block, and it is why a tumbling object throws fragments outward.

The piece's own model data overrides the original's for the transfer factors. That ordering
is the extension point: a specific fragment declares how much of the parent's motion it
inherits.

**Notes** — the piece inherits the original's collision group when there is one. Without that
the fragments of one object would immediately collide with each other at the instant of
separation, which looks like an explosion from inside.

Two optional flags on the original's model data mark the pieces as *small*, or as ignoring
small objects. Small-object handling is a collision-filtering class; the choice is made per
destroyable, not per fragment.

The automatic removal time is read in seconds and stored in milliseconds, with the piece's
own declaration winning over the parent's.

## `PhysicallyRemoveSelf` / `PhysicallyRemovePart`

**Contract** — takes a body out of the world without destroying the object: disable the body,
disable its collision, make it invisible and inactive. The player character is special-cased
and goes through its own removal path, because its body is not a plain shell.

**Notes** — "physically removed" is distinct from destroyed. The object still exists, still
has an identity, and is still removable later by the scheduler poll. That separation is what
makes destruction survivable across a save.

## `SheduleUpdate`

**Contract** — the removal poll, run on the scheduler. Destroys the object outright once it
is destroyed, released (every piece has arrived), permitted by its own veto, and locally
owned. It is a poll rather than an event because the last of those four conditions can
change without notice.

## `Load` (two forms)

**Contract** — reads the debris list. The two forms differ in how much a piece list may say:

- **From the entity's own configuration**: one key naming one debris visual. An object with
  no such key is not destroyable at all.
- **From a model's embedded data**: either that same single key, or — failing that — a whole
  section in which *every key* is a visual name and its value is how many copies of it to
  produce. A non-empty section makes the object destroyable.

**Invariants** — a count of copies is how a crate yields five identical planks from one
declaration. The count defaults to one when the value is empty.

**Notes** — the object's own configuration cannot express the multi-piece form; only model
data can. Nothing explains the restriction.

## `Init` / `RespawnInit`

**Contract** — `Init` only zeroes the outstanding count. `RespawnInit` is the full reset: not
destroyed, released, no debris list, no arrived pieces, nothing outstanding — the state an
object must be in to be reused from a pool.

**Notes** — `RespawnInit` clears the debris list, which `Load` filled. An object reset this
way is no longer destroyable until something loads it again.

## `SetFatalHit` / `FatalHit`

**Contract** — records the blow that will be credited with the destruction: its bone, its
point in that bone's space, its direction and its physical impulse. Held until the pieces
arrive, possibly several frames later, which is why the hit must be a value and not a
reference to anything.
