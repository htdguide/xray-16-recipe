# src/xrGame/ActorBackpack.cpp

> The backpack: a wearable inventory item whose only job is to alter the actor's carry limit and movement.

**Needs** — [`ActorBackpack.h`](ActorBackpack.h.md) · [`Actor.h`](Actor.h.md) · [`Inventory.h`](Inventory.h.md) · [`inventory_item_object.h`](inventory_item_object.h.md)
**Used by** — reached through its declarations in [`ActorBackpack.h`](ActorBackpack.h.md); callers name that, not this file.
**Tier floor** — T3: reads numbers from a section and exposes them

## Purpose

A backpack is an item with no behaviour of its own. It is a bundle of modifiers the
actor's condition and movement code reads off whatever is in the backpack slot. Keeping it
a class rather than a set of loose numbers on the actor is what lets it be an upgradeable,
damageable, tradeable inventory item like any other.

## State

```text
RECORD Backpack
  additional_weight      : real   # raises the carry limit at which overload begins
  additional_weight2     : real   # raises the absolute carry limit
  power_restore_speed    : real   # default 0; added to the actor's stamina regeneration
  power_loss             : real   # default 1; multiplies stamina drain. invariant: in (0, 1]
  jump_speed             : real   # default 1; multiplies jump impulse
  walk_accel             : real   # default 1; multiplies walk acceleration
  overweight_walk_accel  : real   # default 1; multiplies walk acceleration when overloaded
  uses_condition         : bool   # default true; when false the item never wears
```

**Invariants** — the two weight values are the pair every carried-weight limit in the game
is expressed as: the first is where the actor starts to be slowed, the second is where the
actor cannot move at all. A backpack raises both. The stamina-drain multiplier is clamped
away from zero, because zero would make stamina infinite and the clamp is cheaper than
auditing the data.

**Notes** — the class constructor turns condition tracking *off* and the section may turn
it back on. That inversion of the usual default is deliberate: most backpacks in the
shipped data never wear out, and making non-wearing the default means those sections need
no key at all.

## `Load`

**Contract** — reads the section after the base item has. The two weight keys are
required; everything else is optional with the defaults above. A section that omits the
condition key gets condition tracking *on*, overriding the constructor.

## `Hit`

**Contract** — damage taken by the wearer wears the backpack. No effect when condition
tracking is off. The incoming damage is scaled by the item's per-damage-type immunity
before being subtracted from the item's condition, so a backpack tuned to resist burns
degrades slowly in a fire and normally under bullets.

```text
FUNCTION hit(power, damage_type)
  IF NOT uses_condition THEN RETURN
  change_condition(-(power * immunity_for(damage_type)))
```

## `install_upgrade_impl`

**Contract** — applies one upgrade section on top of the loaded values, and in *test* mode
reports whether the upgrade would change anything without applying it. Four of the seven
tunables are upgradeable: the two weight limits and the two stamina modifiers. Movement
modifiers are not, which is a game-balance decision rather than a technical one. The
stamina-drain multiplier is re-clamped after the upgrade, here to the closed unit range
rather than the open one used at load — an inconsistency with no visible consequence,
since no shipped upgrade sets it to zero.
