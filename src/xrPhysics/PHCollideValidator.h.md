# src/xrPhysics/PHCollideValidator.h

> Decides, before any contact is computed, whether two physics objects are allowed
> to touch at all.

**Needs** — [`PHCollideValidator.cpp`](PHCollideValidator.cpp.md) · [`ICollideValidator.h`](ICollideValidator.h.md) · [`PHObject.h`](PHObject.h.md)
**Used by** — [`ActorCameraCollision.cpp`](ActorCameraCollision.cpp.md) · [`IActivationShape.cpp`](IActivationShape.cpp.md) · [`ICollideValidator.h`](ICollideValidator.h.md) · [`PHCollideValidator.cpp`](PHCollideValidator.cpp.md) · [`PHObject.cpp`](PHObject.cpp.md) · [`PHShell.cpp`](PHShell.cpp.md) · [`PHShellActivate.cpp`](PHShellActivate.cpp.md) · [`PHShellSplitter.cpp`](PHShellSplitter.cpp.md) · [`PHSimpleCharacter.cpp`](PHSimpleCharacter.cpp.md) · [`PHStaticGeomShell.cpp`](PHStaticGeomShell.cpp.md) · [`PHWorld.cpp`](PHWorld.cpp.md) · [`Physics.cpp`](Physics.cpp.md)
**Tier floor** — T1: the decision is two bitwise operations evaluated for every candidate
pair in the broad phase, and the layout of the bits is the algorithm.

## Purpose

This is the substance holder of the pair, even though a companion `.cpp` exists: the
*decision rule* is here and only the mutable registry is there. That inversion is
deliberate — the rule must be inlinable, because it runs once per candidate pair per step
and a call would cost more than the test.

Two questions are answered. **May these two objects collide with each other?** and **may
this object collide with the level?** Everything the rest of the chapter does with collision
filtering is expressed by setting bits that these two tests read.

## the filtering model

There are two independent mechanisms, and confusing them is the easiest mistake to make.

**Groups** answer "these specific objects are parts of one thing". Every part of a ragdoll,
a vehicle and its wheels, an object and the thing it was spawned inside, share one group
identifier; two objects in the same group never collide. A group is minted by the game
through [`ICollideValidator.h`](ICollideValidator.h.md) and is an opaque number with no
meaning beyond identity.

**Classes** answer "this kind of thing ignores that kind of thing". Five classes exist —
dynamic, character, small, ragdoll, animated — and membership is orthogonal: one object can
be several. Alongside each class bit sits a *refusal* bit meaning "I do not collide with
members of that class". Refusal is one-sided in its declaration and symmetric in its
effect: if either party refuses the other's class, there is no contact.

```text
RECORD CollisionIdentity                 # carried by every physics object
  group        : int                     # identity; 0 until registered
  class_bits   : set of {
      in_group,                          # "my group id is meaningful"
      no_static,                         # "I never touch the level"
      is_dynamic,     refuses_dynamic,
      is_character,   refuses_character,
      is_small,       refuses_small,
      is_ragdoll,     refuses_ragdoll,
      is_animated,    refuses_animated }
```

**Invariants** — each refusal bit sits immediately above its class bit, so shifting an
object's class bits up by one aligns them with the other object's refusal bits. The whole
class test is then one mask, one shift and one AND per direction. A rebuild is free to use
two separate sets instead; what must survive is that the test is a constant-time bit
operation, not a lookup.

The `in_group` bit is what distinguishes "my group is 0 because I am in group 0" from "my
group is 0 because I have none". Without it every unregistered object would be in one giant
group and nothing would collide with anything.

## `DoCollide`

**Contract** — true when the pair may generate contacts. Pure, constant time, no allocation,
safe to call from the collision pass on any thread.

```text
FUNCTION do_collide(a, b) -> bool
  same_group_pair = a.in_group AND b.in_group
  IF same_group_pair AND a.group == b.group
      RETURN false                       # two parts of one thing
  RETURN NOT ( (a refuses any class b belongs to) OR (b refuses any class a belongs to) )
```

**Invariants** — the group test applies only when **both** objects are group members. An
object with no group is never excluded by the group rule, whatever its identifier field
happens to hold.

The class test is evaluated for both directions, so a refusal declared by either side is
enough. There is no "insistence" bit that could override a refusal — see
[`PHActorCharacter.cpp`](PHActorCharacter.cpp.md) for the one place where a material flag
forces collision back on, and note that it does so *after* this test, by rewriting the
contact rather than by re-admitting the pair.

## `DoCollideStatic`

**Contract** — true when this object may collide with the level's static geometry. One bit
test.

**Notes** — static geometry has no collision identity of its own — it is a triangle soup,
not an object — so the level cannot refuse anybody. The relationship is therefore one-sided
by construction, which is why it needs its own bit rather than a class. Shapes that want to
ignore the level for a different reason (a restrictor cylinder, the camera's anti-character
probe) set a per-*shape* flag instead; see [`GeometryBits.h`](GeometryBits.h.md). The
distinction is per-object here, per-shape there, and both exist because an object can need
one shape that feels walls and another that does not.

## the class and group mutators

**Contract** — the rest of the surface is a set of one-line mutators, all implemented in
[`PHCollideValidator.cpp`](PHCollideValidator.cpp.md): put an object in a class, declare it
refuses a class, put it in a group or in the most recently minted group, take it out of the
dynamic class, forbid it the level, and query whether it is grouped or animated. They are
separate named calls rather than a bit argument because the call sites read as intent — "this
is a ragdoll, it does not collide with characters" — and because the bit layout is private.
