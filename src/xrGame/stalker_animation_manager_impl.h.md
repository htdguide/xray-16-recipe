# src/xrGame/stalker_animation_manager_impl.h

> Five shared predicates the animation channels ask about the stalker's current situation, kept in a header because several channel files need them and none owns them.

**Needs** — [`stalker_animation_manager.h`](stalker_animation_manager.h.md) · [`CharacterPhysicsSupport.h`](CharacterPhysicsSupport.h.md) · [`stalker_movement_manager_smart_cover.h`](stalker_movement_manager_smart_cover.h.md) · [`Weapon.h`](Weapon.h.md) · [`Missile.h`](Missile.h.md) · [`Inventory.h`](Inventory.h.md)
**Used by** — [`stalker_animation_global.cpp`](stalker_animation_global.cpp.md) · [`stalker_animation_legs.cpp`](stalker_animation_legs.cpp.md)
**Tier floor** — T3: five small queries against the movement and inventory subsystems

## Purpose

A header with no implementation file of its own, so it holds its own substance. Its contents
are the questions the torso and legs channels both need answered about the stalker's
situation before they can pick a motion, and they live here rather than in either channel
because both need them and neither is the owner.

## `standing`

**Contract** — is the stalker not locomoting? True when its physical speed is essentially
zero, or when its requested movement type is "stand". No side effects.

```text
FUNCTION standing() -> bool
  IF the physical movement speed is essentially zero  RETURN true
  IF the requested movement type IS stand             RETURN true
  RETURN false
```

**Invariants** — the two conditions are a disjunction and both are needed, because they
disagree in exactly the cases that matter. A stalker that has *asked* to stand but is still
decelerating is standing for animation purposes — it should already be playing the
standing-still set. A stalker asked to walk but pinned against geometry has zero physical
speed and must also play standing-still, or its legs stride in place. Checking only the
request gives the first failure; checking only the speed gives the second.

**Notes** — this is the switch between the two halves of the leg channel, and it is exposed
publicly because the movement system asks it too.

## `body_state`

**Contract** — the stalker's current posture — standing or crouching — read from the movement
system. It is the first index into the animation table, so every selection begins here.

## `fill_object_info`

**Contract** — classify the item currently in the stalker's hands as a weapon, a thrown
item, or neither, and cache both narrowings for the frame. Hard-fails if nothing is in hand.

**Notes** — the classification is cached per frame rather than asked per selection because
the torso channel asks it several times while resolving one motion, and each ask is a
run-time type narrowing. The hard failure on an empty hand is the caller's contract: the
torso channel checks for an empty hand first and takes a different path.

## `object_slot`

**Contract** — the animation slot of whatever is in hand — the small integer that selects
which family of hold-and-aim motions a given item uses — or zero when the hand is empty or
holds something with no animation of its own.

**Invariants** — zero means "no item", and it is a real index in the table rather than a
sentinel, so an item that fails to classify falls back to the empty-handed animations rather
than to an invalid lookup. That is why every motion family has an entry at zero.

## `strapped`

**Contract** — is the weapon in hand slung rather than held ready? Asked of the item-handling
subsystem, not of the weapon, because the answer depends on the transition currently
playing. Hard-fails if nothing is in hand.

**Notes** — slung and ready are different motion families for the same weapon, and the answer
has to come from the handler because a weapon mid-sling is neither: the handler knows which
end of the transition to animate toward.
