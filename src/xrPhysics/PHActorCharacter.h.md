# src/xrPhysics/PHActorCharacter.h

> The player's character controller, and the mutual-exclusion volumes that keep
> creatures of different sizes at sensible distances from the player and from each other.

**Needs** — [`PHActorCharacter.cpp`](PHActorCharacter.cpp.md) · [`PHSimpleCharacter.h`](PHSimpleCharacter.h.md) · [`PHActorCharacterInline.h`](PHActorCharacterInline.h.md) · [`PHCharacter.h`](PHCharacter.h.md) · [`ExtendedGeom.h`](ExtendedGeom.h.md) · [`PhysicsExternalCommon.h`](PhysicsExternalCommon.h.md) · [`IPhysicsShellHolder.h`](IPhysicsShellHolder.h.md)
**Used by** — [`PHActorCharacter.cpp`](PHActorCharacter.cpp.md) · [`PHCharacter.cpp`](PHCharacter.cpp.md)
**Tier floor** — T1: the restrictor is a shape attached to a body through a transform node,
and its contact rule runs inside the solver's collision pass.

## Purpose

Declares the player's controller, whose substance is in
[`PHActorCharacter.cpp`](PHActorCharacter.cpp.md). It is also the only place the
**restrictor** is defined, and that idea does not appear anywhere else, so it is explained
here.

## the restrictor

A character's collision capsule is as wide as its body. That is correct for walls and wrong
for other creatures: two stalkers whose capsules merely touch are standing closer than any
animation can sell, and a small creature and a large one need different spacing. So the
player's controller carries, in addition to its capsule, a set of *restrictors* — upright
cylinders attached to the same body, one per size class, each with its own radius, which do
not collide with the world and exist only to be felt by other characters.

```text
RECORD Restrictor
  type      : RestrictionType     # which size class this cylinder represents
  character : Character           # the owner; none until created
  shape     : Cylinder            # radius = spacing for this class, height = the body's
  radius    : real                # default 0.1 until the game sets it from data
```

**Invariants** — the cylinder is offset upward by half the body height so it stands on the
ground rather than straddling it, and it is explicitly marked as not colliding with static
geometry. A restrictor that felt walls would stop the player a restrictor-radius short of
every doorway.

The player's controller creates exactly three restrictors — large stalker, small stalker,
medium monster — and they are indexed by the size-class value itself, so the enumeration's
order is load-bearing: the restriction types below `actor` must be contiguous and start at
zero. This is asserted at every lookup.

## the restrictor contact rule

**Contract** — when a restrictor cylinder touches something, the contact is suppressed by
default and reinstated only if the other side agrees.

```text
FUNCTION restrictor_contact(contact, i_am_first, my_type)
  suppress the contact
  IF either side has no body, no user data or is not a CHARACTER    RETURN
  mine, theirs = the two characters, ordered so `mine` owns this restrictor
  mine.choose_restriction_type(my_type, contact.depth, theirs)   # may resize the spacing
  reinstate the contact only if `theirs` accepts being held off by a restrictor of my_type
```

**Invariants** — the other character decides whether it is held off. That is what makes the
restrictor a *negotiation* rather than an obstacle: a creature can declare that it ignores
stalker spacing (a blind dog does not care), and a creature currently being pushed by the
player can drop the restriction to let itself be moved.

**Notes** — the contact also carries its penetration depth into the size-class decision, so
the owner knows not merely that another creature is near but how far inside the spacing it
has come. Only the large-stalker restrictor uses that, and only to decide whether to swap
itself for the small one — see [`PHActorCharacter.cpp`](PHActorCharacter.cpp.md).

The whole mechanism is per-class rather than per-pair because the shipped data expresses
spacing as a property of a creature's size, not of a relationship. A rebuild that wants
per-pair spacing would replace the fixed three cylinders with one whose radius is chosen in
the callback, which the structure here already almost permits.

## `CPHActorCharacter`

**Contract** — the concrete controller for the player, on top of the shared simple
controller. Beyond the base surface it adds: per-class restrictor radii settable from game
data, a single-player/multiplayer mode flag fixed at construction, and overrides for
creation, destruction, material assignment, acceleration, jumping, disabling, contact
initialisation, and the walk-on validation that decides whether a surface can be climbed.
All of it is described in [`PHActorCharacter.cpp`](PHActorCharacter.cpp.md).

**Notes** — the mode flag is not a runtime switch. Player-versus-player collision rules,
restrictor use and contact separation all differ between the two modes, and the choice is
made by the factory in [`PHCharacter.cpp`](PHCharacter.cpp.md) when the controller is
built.
