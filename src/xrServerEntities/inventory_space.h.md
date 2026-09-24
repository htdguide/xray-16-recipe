# src/xrServerEntities/inventory_space.h

> Where a carried thing can be: the numbered equipment slots, the three places an item can live, and the packed field that records the one it is in.

**Needs** — _(none of substance)_
**Used by** — [`HudItem.cpp`](../xrGame/HudItem.cpp.md) · [`HudItem.h`](../xrGame/HudItem.h.md) · [`InventoryBox.h`](../xrGame/InventoryBox.h.md) · [`InventoryOwner.h`](../xrGame/InventoryOwner.h.md) · [`UIGameCustom.h`](../xrGame/UIGameCustom.h.md) · [`game_sv_deathmatch.h`](../xrGame/game_sv_deathmatch.h.md) · [`inventory_item.h`](../xrGame/inventory_item.h.md) · [`UIActorMenu.h`](../xrGame/ui/UIActorMenu.h.md) · [`UIDragDropReferenceList.h`](../xrGame/ui/UIDragDropReferenceList.h.md) · [`UIHudStatesWnd.h`](../xrGame/ui/UIHudStatesWnd.h.md) · [`xrServer_Objects_ALife_Items.h`](xrServer_Objects_ALife_Items.h.md)
**Tier floor** — T1: the placement field is a packed 16-bit word written into entity records.

## Purpose

An inventory item's record has to say *where* the item is — which slot, or which of the two
loose containers — and this file defines that vocabulary. The slot numbering is frozen by
shipped configuration (every item section names its slot by number) and by the key bindings
(the number keys select slots by index).

## State

```text
ENUM Slot : int
  none = 0
  knife = 1, secondary = 2, primary = 3, grenade = 4,
  binocular = 5, bolt = 6,
  outfit = 7, pda = 8, detector = 9, torch = 10,
  artefact = 11, helmet = 12, backpack = 13
  count = 14

ENUM Place : int
  undefined = 0
  slot      = 1      # equipped in one of the numbered slots
  belt      = 2      # on the quick-access belt
  rucksack  = 3      # carried but not to hand

RECORD ItemPlacement                 # exactly 16 bits, and the packing is the contract
  place         : int (4-bit)        # a Place value
  slot          : int (6-bit)        # the slot currently occupied
  base_slot     : int (6-bit)        # the slot this item belongs in
```

**Invariants**

- **Slot zero means "no slot"**, so the first real slot is 1. The comment trail in the
  source records that this was once zero-based and was shifted; every shipped configuration
  uses the current numbering.
- The placement word is sixteen bits split 4/6/6 and is written to records as one value.
  Six bits caps the slot count at 63, comfortably above the 14 in use.
- `slot` and `base_slot` differ when an item is temporarily displaced — a weapon moved aside
  by a scripted sequence keeps the slot it *belongs* in, so it can go back.

## the rucksack grid

```text
CONSTANT rucksack_width  = 7     # cells across
CONSTANT rucksack_height = 280   # cells down
```

**Notes** — the width is a real user-interface constraint: every item's configuration
declares an icon footprint in grid cells, and seven is how many fit across the panel. The
height is effectively unbounded — 280 rows is "as much as you will ever carry", not a
designed limit — and the real cap on carrying is weight, enforced elsewhere.

## the inventory-blocking flags

A small set of named bit values says why the inventory is currently unusable: on a ladder,
in a vehicle, blocked entirely, the inventory window is open, the buy menu is open. They are
assigned at startup rather than being constants, because the script layer can add its own.
Only the fact that they are a bitmask matters to this chapter.

## the brief-info record

**Contract** — a bundle of already-formatted display strings for one item (name, icon,
per-ammo-type counts, fire mode, grenade state) that an item fills in on request for the
heads-up display. Presentation only; it never reaches a record or the wire. It is in this
header because the item records are the things asked to fill it.
