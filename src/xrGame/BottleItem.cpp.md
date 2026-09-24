# src/xrGame/BottleItem.cpp

> A drinkable that shatters: hit it hard enough and it breaks, with a sound and a particle burst, and ceases to exist.

**Needs** — [`BottleItem.h`](BottleItem.h.md) · [`FoodItem.h`](FoodItem.h.md) · [`entity_alive.h`](entity_alive.h.md) · [`EntityCondition.h`](EntityCondition.h.md) · [`xrMessages.h`](../xrServerEntities/xrMessages.h.md) · [Seam: Audio device](../../SYSTEM-REQUIREMENTS.md#seam-audio-device)
**Used by** — reached through its declarations in [`BottleItem.h`](BottleItem.h.md); callers name that, not this file.
**Tier floor** — T3: an event, a sound and a particle effect

## Purpose

A bottle is a consumable with one extra property: it is fragile. The file's whole content is
the breakage path, and its interest is entirely in *how* that path is routed — the breakage
is not done where it is detected.

## State

```text
  break_particles : optional<text>    # the shatter effect; absent means no visual
  break_sound     : sound handle      # created at load, destroyed with the item
  BREAK_POWER     : real = 5.0        # damage above which the bottle breaks
```

## `Load`

**Contract** — reads the optional particle-effect name and the optional shatter sound. Both
absent is legal: a bottle with no configured effects breaks silently and invisibly.

## `Hit`

**Contract** — after the base class has applied the damage, if the damage exceeded the
threshold, the bottle **sends itself an explosion event** rather than breaking directly, and
only from the authoritative side.

**Invariants** — routing through an event is the chapter's rule and here it earns its
keep twice over: the breakage must be seen by every client, and breaking an object from
inside its own damage handler would destroy it while the damage path still holds it.

**Notes** — the event reused is the grenade explosion message. That is a piggyback on an
existing message rather than a new one, and a rebuild defining its own protocol should give
breakage its own message; nothing about the grenade message's payload is used.

The threshold is a compiled-in constant, not configuration, so every bottle in the game has
the same fragility.

## `OnEvent`

**Contract** — on the explosion message, shatter. Every other message goes to the base class
only.

## `BreakToPieces`

**Contract** — plays the shatter sound at the bottle's position, spawns the shatter particle
effect there if one is configured, and — only on the authoritative side — destroys the
object.

**Invariants** — the sound and the particles are played on **every** machine, and only the
destruction is gated on ownership. That split is the general pattern for a destructive
effect in this chapter: presentation everywhere, state change once.

**Notes** — the particle object is created detached and plays itself out; nothing owns it
after creation, which is correct because the object that spawned it is about to be
destroyed.
