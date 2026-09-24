# src/xrGame/doors_door.cpp

> One door: the two positions its leaf can be in, a claim list of the creatures that want it open or shut, and the rule that restores it to how it was found once they have all gone.

**Needs** — [`doors_door.h`](doors_door.h.md) · [`doors.h`](doors.h.md) · [`doors_actor.h`](doors_actor.h.md) · [`PhysicObject.h`](PhysicObject.h.md) · [`script_game_object.h`](script_game_object.h.md) · [Seam: Script binding layer](../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: a small state machine with a claim list

## Purpose

The engine does not open doors. A door is a physics object animated by a script binder, and
all this record does is decide *which way it should be* and ask the script to make it so.
That indirection is the design: door animation, sound and locking behaviour are content, and
the engine owns only the arbitration between several creatures wanting different things from
the same door at the same time.

## State

```text
RECORD Door
  object              : PhysicObject     # the thing in the world
  state               : DoorState        # where it actually is, per the world's reports
  target_state        : DoorState        # where the claimants want it
  previous_state      : DoorState        # where it was before anyone claimed it
  initiators          : list<Agent>      # the creatures currently claiming it
  open_vector         : vector           # the leaf's tip when open, in object-local space
  closed_vector       : vector           # the leaf's tip when closed, likewise
  registered_position : vector           # the hinge, captured once at registration
  locked              : bool
```

**Invariant** — `previous_state` is meaningful only while the claim list is non-empty. It is
captured the moment the door arrives somewhere with nobody claiming it, and consumed when
the last claimant leaves. This is what makes the subsystem *polite*: a creature that opens a
door behind itself leaves the world as it found it, and a door the level author shipped open
does not end the game shut.

**Invariant** — `registered_position` is the position at registration and is never updated,
while `get_matrix` returns the object's live transform. Both are correct for their uses: the
spatial index needs a fixed key, and the geometry tests need the current pose. Confusing them
would either corrupt the index or test against a stale swing.

**Invariant** — the two swing vectors are in the *object's local* space and are scaled to the
leaf length. They are captured once at construction, from the pose the door's authored
skeleton has, and never recomputed. So a door whose authored geometry changes at runtime —
which nothing does — would keep the old swing.

## Construction

**Contract** — takes the physics object, starts in the open state on all three state fields,
captures the hinge position, and derives the two swing vectors. Requires the object to expose
door vectors; a physics object without a bone named for a door and a box shape on it cannot
be a door, and this is a hard failure rather than a refusal.

```text
FUNCTION construct(object)
  state = target_state = previous_state = open
  registered_position = object.position
  locked = false
  (closed_vector, open_vector) = object.door_vectors()      # world space
  invert the object's transform and rotate both into object-local space
  scale both to the leaf length (1.1)
  mark the object as visible to the AI's spatial queries
```

**Invariants** — the vectors arrive in world space and are rotated — not transformed — into
local space, because they are directions from the hinge, not positions. Scaling them to a
fixed leaf length rather than their real length means every door in the game is treated as
having the same size leaf; that is a simplification the AI geometry relies on.

**Invariants** — initializing all three states to *open* is an assumption, not an
observation. Nothing has asked the world yet. It is harmless because the first arrival report
corrects both the current and the previous state, and because a door nobody has claimed does
not act on its target.

**Notes** — the object is flagged as visible to the AI's spatial system at construction and
unflagged at destruction. Without that flag the door is invisible to the queries that build
the AI's picture of its surroundings.

## Destruction

**Contract** — clears the AI-visibility flag and notifies every current claimant that the
door is gone, so they can drop their references. Order matters: the claimants hold handles
into this record.

## `change_state(initiator, state)` — the one verb

**Contract** — the public entry point, forwarding to the three-argument form with the
opposite state as the stop state. Its meaning depends on the door's current target, and this
overloading is the single most important thing on the page:

```text
FUNCTION change_state(initiator, start_state, stop_state)
  IF nobody is claiming the door THEN
    # first claim: adopt the requested state and ask the script to move
    claim(initiator) ; target_state = start_state ; request_move(initiator) ; RETURN

  IF target_state == start_state THEN
    # somebody already wants what I want: just join the claim
    claim(initiator) ; RETURN                # requires I am not already claiming

  # target_state == stop_state: this call is a RELEASE, not a claim
  remove initiator from the claim list      # it must be there
  IF anyone still claims it THEN RETURN
  IF previous_state != stop_state THEN
    target_state = previous_state ; request_move(none)
```

**Invariants** — a claim and a release are the *same call with the same arguments*; which one
happens is decided by whether the door is already trying to do what you asked. A creature
asks for "open" to claim and asks for "open" again — via the release path, with the door now
targeting closed — to let go. The creature agent in [`doors_actor.cpp`](doors_actor.cpp.md)
is built around this and calls it with the *stop* state to release.

**Invariants** — a creature may claim a door only once. A double claim is a contract
violation, because the release path removes exactly one entry and a doubled claimant would
pin the door forever.

**Invariants** — when the last claimant leaves, the door returns to `previous_state` — the
state it was in before anyone touched it — and not to a default. A door authored open stays
open.

**Notes** — the restore is skipped when the previous state already equals the stop state,
which is the case where releasing leaves the door where it was anyway.

## `change_state(initiator)` — asking the script to move

**Contract** — the private request. Does nothing when the door is already at its target, and
nothing when the object is not spawned. Otherwise it fires the object's **use** callback into
the script layer, passing both the door and the creature that wants it.

**Invariants** — this is the *only* place the door is made to move, and it does so by
invoking a script callback. The engine has no animation, no sound and no timing for doors;
the content does. A rebuild must keep the callback and its two arguments, because the shipped
door binder scripts read them.

**Notes** — the initiator is passed so that scripts can react to *who* is opening the door —
a locked door refusing a faction, a trap triggering on the player. It is optional, and the
restore-on-release path passes nothing, because no one is opening it: it is closing itself.

## `on_change_state`

**Contract** — the world's report that the door arrived at a state. Records it as the current
state. Then one of two things:

```text
FUNCTION on_change_state(state)
  state = state
  IF nobody is claiming the door THEN
    previous_state = state         # this is now "how the world found it"
    RETURN
  request_move(none)               # still not where the claimants want it? ask again
```

**Invariants** — an unclaimed door's arrival *defines* the previous state. That is how a door
the player opens by hand becomes the state creatures will later restore it to.

**Invariants** — a claimed door's arrival re-triggers the move request, which is a no-op when
it arrived where it was asked and a retry when it did not. A door that a script moved the
wrong way — which the source notes does happen — is corrected rather than left inconsistent.

## `is_locked` · `lock` · `unlock`

**Contract** — `is_locked(state)` is **not** simply the lock flag: a locked door that is
already in the state you want is reported as *not* locked, because it is not in your way.
Locking and unlocking are idempotence-checked — locking a locked door is a contract violation
rather than a no-op, which catches a script that locks without tracking.

## `is_blocked`

**Contract** — true when somebody is claiming the door and their target is not the state you
want. An unclaimed door is never blocked. This is the contention test: two creatures wanting
opposite things, where the first to claim wins and the second waits.

## `position` · `get_matrix` · `get_vector`

**Contract** — the registered hinge position (for the spatial index), the object's live
transform (for geometry tests), and the leaf tip in a given state, in object-local space.
Callers transform the vector by the matrix to get the leaf's world segment.

## Debug surface

**Contract** — the door's authored name, a comma-joined list of the names of every current
claimant, and a membership test. The claimant list is the only way to answer "why is this
door not opening", which is why it is assembled at all.

**Notes** — the list is built into a stack buffer sized by a first pass over the names. It is
a debug convenience with no counterpart in a rebuild.
