# src/xrPhysics/PHCaptureInit.cpp

> Deciding whether a grab is possible at all, reading its parameters out of the
> creature's own data, and unwinding it cleanly.

**Needs** — [`PHCapture.h`](PHCapture.h.md) · [`PHCapture.cpp`](PHCapture.cpp.md) · [`PHCharacter.h`](PHCharacter.h.md) · [`PHElement.h`](PHElement.h.md) · [`PhysicsShell.h`](PhysicsShell.h.md) · [`PHWorld.h`](PHWorld.h.md) · [`PhysicsCommon.h`](PhysicsCommon.h.md) · [`ExtendedGeom.h`](ExtendedGeom.h.md) · [`IPhysicsShellHolder.h`](IPhysicsShellHolder.h.md) · [`Include/xrRender/Kinematics.h`](../Include/xrRender/Kinematics.h.md) · [`xrCore/Animation/Bone.hpp`](../xrCore/Animation/Bone.hpp.md)
**Used by** — [`PHCapture.cpp`](PHCapture.cpp.md) · [`PHCapture.h`](PHCapture.h.md)
**Tier floor** — T2: predicates and configuration reads; the one physical act is setting a
velocity limit.

## Purpose

The construction and destruction half of a capture, split out from the per-step half in
[`PHCapture.cpp`](PHCapture.cpp.md). What lives here is the *eligibility* rules and the
*data-driven* parameters — the parts a designer changes — and the release path.

The separation from the other file is by lifecycle phase, not by concern, and there is no
reason a rebuild must keep it.

Stateless beyond the capture record declared in [`PHCapture.h`](PHCapture.h.md).

## can this creature capture this object

**Contract** — a predicate, checked before anything is built. Every clause is a veto.

```text
FUNCTION can_capture(capturer, target) -> bool
  target must exist, have an ACTIVE physics shell, and not be an inventory item
  capturer must exist, have a body, and have a game object with a SKELETON
  the capturer's skeleton data must declare a `capture` section
  RETURN true
```

```text
FUNCTION can_capture(capturer, target, named_part) -> bool
  can_capture(capturer, target)
  AND the named part is a real bone of the target's skeleton
  AND that bone is bound to a physics body
```

**Invariants** — the `capture` section in the capturing creature's skeleton data is the
feature switch for the entire mechanism. A creature whose model carries no such section
cannot capture, and nothing else needs to be configured to disable it. That places the
decision with the art data rather than with code or with the creature's class, which is why
a mod can give a new creature telekinesis without touching the engine.

Inventory items are excluded outright. They have physics shells, but picking one up is the
inventory system's job and a physical grip would fight it.

The named-part form additionally requires the target bone to *be* a physics body. In a
partly-physical skeleton only some bones are; asking to grab an animated one is a request
the mechanism cannot satisfy, and it is refused rather than approximated.

## choosing the target body

**Contract** — two ways in, differing only here.

- **By proximity** — the target's shell is asked for the body nearest the capture bone,
  optionally filtered by a caller-supplied predicate so a creature can refuse particular
  parts.
- **By named part** — the caller names a bone and the body bound to it is taken directly.

**Notes** — the proximity form is what a creature's brain uses when it decides to grab "that
crate"; the named form is used when the game already knows which part it wants — most often
by a script. Neither form fails loudly: a capture that cannot resolve a target body simply
stays in the free state, and the caller discovers this through the published failure query.

## the capture bone

**Contract** — the bone the grip hangs from is named in the `capture` section of the
capturing creature's skeleton data and resolved to a bone instance at construction. A name
that does not resolve is a hard failure.

**Notes** — a bone *instance* is taken, not a bone index, because the capture reads the
bone's animated transform every step. The instance is owned by the creature's skeleton, so
the capture holds a borrowed reference whose lifetime is the creature's — one more reason
the destruction path in [`PHCapture.cpp`](PHCapture.cpp.md) must run before the creature
goes away.

## the parameters

**Contract** — read from the capturing creature's `capture` section at construction.

```text
pull_distance   : real   # beyond this, the grip is abandoned
distance        : real   # within this, the pull becomes a hold
capture_force   : real   # the grip's strength
time_limit      : int    # seconds, converted to milliseconds
pull_force      : real   # an upper bound; see below
velocity_scale  : real   # how much the held object's speed cap is reduced
```

```text
FUNCTION init()
  IF the target is already beyond pull_distance   RETURN     # never starts
  pull_force = min( 4 * gravity * the target shell's mass,   the authored pull force )
  lower the target's linear and angular velocity limits by `velocity_scale`
  install the contact-suppression callback on the CAPTURER
  initialise this capture's island
  IF the capturer is the player, hide its weapons
  activate as a per-step object ; state = pulling
```

**Invariants** — the pull force is *mass-relative with an authored ceiling*: four times the
weight of the thing being pulled, but never more than the creature is allowed. The
mass-relative part is what makes the pull look the same on a can and on a crate — both
accelerate at the same rate — and the ceiling is what makes a genuinely heavy object refuse
to come. A rebuild that uses the authored force directly will find light objects flying and
heavy ones immobile.

The target's speed limits are reduced for the duration of the pull and restored at the
moment of capture. This is the only protection against the pulled object arriving at speed;
the pull force alone would happily accelerate a light object across a room.

**Notes** — the time limit is authored in seconds and stored in milliseconds. It bounds a
pull that is never going to arrive — the object is snagged on geometry, or the creature is
walking backwards — and releasing on it is the difference between a creature that gives up
and one that drags a crate around a corner forever.

Hiding the player's weapons is the only game-layer effect in this file, and it is here
because the capture is the authority on when the grip starts and ends. A rebuild should
raise this to an event the game layer subscribes to, rather than a call physics makes.

## `Release`

**Contract** — voluntary let-go. Idempotent: calling it in the released or free state does
nothing.

```text
FUNCTION release()
  IF already released or free    RETURN
  remove and destroy the ball joint, the motor joint and the anchor body,
      taking each out of my island first
  IF we were still PULLING and the target is alive
      restore the target's default velocity limits          # see Notes
  mark as still colliding                                   # forces one more settle step
  IF the capturer is the player, show its weapons again
  state = released
```

**Invariants** — the velocity limits are restored here only on the pulling path, because the
captured path already restored them at the moment of capture. Missing this leaves an object
that was released mid-pull permanently slow, and the effect survives saving and loading,
because the limit is part of the element's state.

Setting the collided flag on release is what guarantees at least one step in the released
state before the capture goes free. Without it, a release in a step where nothing happened to
touch would go free immediately and the contact suppression would end while the object was
still inside the creature.

## `Deactivate`

**Contract** — the hard unwind, used by the destructor and by the severing path. Releases,
wakes the target so it does not stay asleep holding a stale pose, clears the capturer's
contact callback, stops per-step updates, and null the three participant references.

**Invariants** — clearing the capturer's contact callback is mandatory and is the mirror of
installing it. A creature left with a capture's callback after the capture is gone suppresses
contacts against an object it no longer holds.
