# src/xrGame/inventory_item.h

> Declares the mix-in every carryable thing in the game inherits: a name, a weight, a cost, a condition, a place in a grid, an upgrade list, and a network-synchronized physical body.

**Needs** — [`inventory_space.h`](../xrServerEntities/inventory_space.h.md) · [`attachable_item.h`](attachable_item.h.md) · [`hit_immunity.h`](hit_immunity.h.md) · [`inventory_item_inline.h`](inventory_item_inline.h.md) · [`xrServer_Objects_ALife_Items.h`](../xrServerEntities/xrServer_Objects_ALife_Items.h.md)
**Used by** — [`Explosive.h`](Explosive.h.md) · [`ExplosiveRocket.cpp`](ExplosiveRocket.cpp.md) · [`ExplosiveRocket.h`](ExplosiveRocket.h.md) · [`HUDTarget.cpp`](HUDTarget.cpp.md) · [`HudItem.cpp`](HudItem.cpp.md) · [`Inventory.cpp`](Inventory.cpp.md) · [`Inventory.h`](Inventory.h.md) · [`InventoryBox.cpp`](InventoryBox.cpp.md) · [`WeaponUpgrade.cpp`](WeaponUpgrade.cpp.md) · [`ai_stalker_alife.cpp`](ai_stalker_alife.cpp.md) · [`attachable_item.cpp`](attachable_item.cpp.md) · [`attachable_item.h`](attachable_item.h.md) · [`attachment_owner.cpp`](attachment_owner.cpp.md) · [`eatable_item.h`](eatable_item.h.md) · _and 16 more_
**Tier floor** — T2: a declaration

## Purpose

Declares the surface implemented in [`inventory_item.cpp`](inventory_item.cpp.md),
[`inventory_item_upgrade.cpp`](inventory_item_upgrade.cpp.md) and
[`inventory_item_inline.h`](inventory_item_inline.h.md). Every weapon, artefact, outfit,
medkit, bolt and quest token in the game carries one of these.

Two pieces of substance live here rather than in any implementation file, because they are
declarations of shape.

## State

```text
ENUM HandDependence  none | one_handed | two_handed

RECORD ItemFlags                 # a 16-bit set; the ORDER of bits is not load-bearing
  drop_manual        # queued for dropping; see the implementation's note on trade
  can_take           # may be picked up at all
  can_trade          # may be traded right now
  belt               # currently on the belt
  ruck               # currently in the backpack
  ruck_default       # goes to the backpack rather than a slot when picked up
  using_condition    # this item wears out
  allow_sprint       # the owner may sprint while it is active
  useful_for_npc     # creatures will pick it up
  in_interpolation   # network interpolation is running
  in_interpolate     # (set nowhere in the shipped build)
  is_quest_item      # never tradeable, never dropped
  is_helper_item     # engine-created, not part of the player's belongings

RECORD NetworkSample
  timestamp : int
  state     : PhysicsState    # position, orientation, velocities, force, torque

RECORD NetworkData
  samples    : queue<NetworkSample>   # at most two: interpolation needs a span
  start_time : int
  end_time   : int
```

**Invariants** — the trade permission is *two* pieces of state, not one: a fixed
"this section may ever be traded", read from configuration, and a current
"may be traded right now" flag. Restoring permission restores it to the fixed value, so a
temporarily untradeable item cannot be made tradeable by accident. That distinction is the
only non-obvious thing in the flag set.

The belt and backpack flags are not exclusive with the slot placement — the authoritative
placement is the separate place record, and these are cached answers.

## Exported units

- **Identity and description** — the section, the translated full and short names, the
  description, the inventory grid rectangle, the icon, the upgrade-icon rectangle and the
  kill-message rectangle. All from configuration; the translated ones can be reloaded when the
  language changes.
- **Economics** — weight, cost, condition, and the wear rules.
- **Placement** — the current place, the base slot, the belt and backpack flags, and three
  move notifications with empty defaults.
- **Permissions** — may be taken, may be traded, is a quest item, allows sprinting, is useful
  to creatures.
- **Lifecycle** — construct, load tuning, reload tuning, reinitialize, spawn from a server
  record, destroy, and the four owner-transition hooks.
- **Persistence** — save and load.
- **Network** — export, import, interpolate, and the four physics correction-prediction hooks.
- **Attachment** — attach, detach, and their permission queries; all refuse by default.
- **Upgrades** — the installed list, the queries over it, installation and verification. See
  [`inventory_item_upgrade.cpp`](inventory_item_upgrade.cpp.md).
- **Lethality queries** — five of them, all answering "no" here, overridden by weapons; they
  exist so that a creature's planner can ask any item whether it could kill with it.
- **A family of downcasts** — one per item kind. This is the engine's type-dispatch mechanism
  in the inventory layer: rather than a type tag, an item answers a question per kind and
  returns itself or nothing. A rebuild with real type queries needs none of them, but the
  *set* of kinds is informative: eatable, weapon, food, missile, held item, ammunition,
  attachable, physics holder, game object.
- **Holder modifiers** — multipliers this item applies to its owner's view range and field of
  view while it is the active one.

**Notes** — two helper templates read a configuration key only if it exists and then either
add to or replace a value, with a "test only" mode that checks presence without writing.
They exist for the upgrade system, where the same pass must first validate that every key an
upgrade names exists and then apply them. Both are in
[`inventory_item_inline.h`](inventory_item_inline.h.md).
