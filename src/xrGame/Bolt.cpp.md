# src/xrGame/Bolt.cpp

> The bolt: a throwable that is never consumed, used to probe for anomalies.

**Needs** — [`Bolt.h`](Bolt.h.md) · [`Missile.h`](Missile.h.md) · [`ParticlesObject.h`](ParticlesObject.h.md) · [`xrPhysics/PhysicsShell.h`](../xrPhysics/PhysicsShell.h.md) · [`xrPhysics/DamageSource.h`](../xrPhysics/DamageSource.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: a thrown item with two tuning overrides

## Purpose

The bolt is the series' signature tool: you throw it ahead of you and watch whether an
anomaly reacts. Mechanically it is an ordinary thrown item with three differences, and each
is one line here.

**It is infinite.** Throwing a bolt immediately spawns its replacement in the player's hand,
so the item in the inventory is never consumed. That is why the player is given one bolt and
never needs another.

**It is never useful.** The bolt declares itself not worth picking up, so the pickup logic
in [`Actor_Feel.cpp`](Actor_Feel.cpp.md) and every automatic-pickup path ignores thrown
bolts lying on the ground. Without this the world would fill with bolts the player keeps
picking up.

**It remembers who threw it.** An anomaly triggered by a bolt attributes the trigger to the
thrower, which is what lets an anomaly's damage and its script callbacks name a culprit.

## State

```text
  thrower_id : entity identifier   # who threw it; invalid until it is taken
```

## `OnH_A_Chield`

**Contract** — when the bolt becomes a child of something, the thrower is recorded as its
*grandparent* — the bolt is held by a hand which belongs to a creature. Runs after the base
class's own become-a-child handling.

**Notes** — reaching two levels up the ownership chain is the convention for "who is
ultimately responsible for this item" throughout the chapter.

## `Throw`

**Contract** — throws the bolt and immediately creates a replacement in hand. The flying
bolt is given a lifetime derived from the maximum destroy time **divided by the physics time
factor**, so that a bolt thrown during accelerated or slowed time still exists for the same
number of simulated seconds.

**Invariants** — the replacement is spawned *after* the base throw, so the throw operates on
the real object and the replacement never participates in it.

**Notes** — the flying object is a separate "fake missile" from the inventory item, which is
the base thrown-item mechanism. The bolt's contribution is only that it re-spawns the fake
immediately rather than when the player next throws.

## `Useful`

**Contract** — always false. See Purpose.

## `activate_physic_shell`

**Contract** — after the base class creates the flying body, sets an almost-zero air
resistance, so a bolt flies a long flat arc rather than dropping like a stone. This is the
one number that decides how far the player can probe.

## `SetInitiator` · `Initiator`

**Contract** — the thrower's identifier, written when the bolt is taken and read by anything
an anomaly attributes to it.

## `Action`

**Contract** — delegates to the base thrown-item action handling and consumes nothing of its
own. A disabled block in the source implemented a separate charge-and-release throw on the
drop action; a rebuild should omit it.

## Notes

The bolt declares itself a damage source and declares that it does **not** use navigation
positions. The second matters: a thrown bolt is not placed on the navigation mesh, so the
pathfinder and the cover system never see it. Every loose item that is purely physical
should say the same.
