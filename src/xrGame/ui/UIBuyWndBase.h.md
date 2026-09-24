# src/xrGame/ui/UIBuyWndBase.h

> The interface every multiplayer buy menu must satisfy, and the "preset" — a saved loadout
> the player can recall between rounds.

**Needs** — [`UIDialogWnd.h`](UIDialogWnd.h.md)
**Used by** — [`UIGameCTA.cpp`](../UIGameCTA.cpp.md) · [`game_cl_deathmatch.h`](../game_cl_deathmatch.h.md) · [`UIMpTradeWnd.h`](UIMpTradeWnd.h.md)
**Tier floor** — T2: an interface over a screen that mutates game state

## Purpose

An interface header, and therefore substantive: what it demands of an implementor *is* the
contract a rebuild must satisfy. It exists because the buy menu was reimplemented — the
original and its replacement both had to be reachable from the same call sites — and the
surface it abstracts is unusually wide, which is itself the information: **the buy menu is not
a screen the game reads from, it is a screen the game drives.**

Only one implementation ships
([`UIMpTradeWnd.h`](UIMpTradeWnd.h.md)); the choice is made by a build-time switch in
[`UIBuyWndShared.h`](UIBuyWndShared.h.md).

## `ETradePreset` — the seven preset slots

```text
ENUM TradePreset
  last        # the loadout the player bought last round
  slot_1      # three player-saved loadouts
  slot_2
  slot_3
  origin      # what the player started this round holding
  temp        # scratch, used while building a purchase
  default     # the game type's authored starting kit
```

**Invariants** — Seven slots, fixed. Three of them are player-visible (`slot_1`…`slot_3`); the
other four are the machinery: `last` is what "buy the same again" recalls, `origin` is what a
cancel restores, `temp` is where a purchase is assembled before it is committed, and `default`
is the fallback when a player has saved nothing. The enumeration's order is frozen because the
presets are persisted by index.

## `_preset_item` — one line of a saved loadout

```text
RECORD PresetItem
  section     : text        # the item's configuration section
  count       : int         # how many
  addon_state : int (8-bit, bit flags)   # which attach points are filled
  addon_names : text [3]    # silencer, scope, grenade launcher, by section
```

**Invariants** — A preset stores **sections and counts, not items**. Recalling a preset is a
purchase, not a restoration: the items are bought fresh at the current prices and the current
rank's availability, so a preset can fail to recall in full. The three attachment slots mirror
the closed enumeration in [`UICellCustomItems.h`](UICellCustomItems.h.md); a weapon's
attachments are part of its preset line, not separate lines.

Equality is by section alone, which is what lets a preset line be found and its count updated.

## `IBuyWnd` — the interface

**Contract** — Every operation below must be provided. Grouped by what the caller is doing.

### Construction

- `Init(section, price section)` — build the menu against a named item table and a named price
  table. Both are configuration section names; the menu's contents are entirely data.
- `BindDragDropListEvents(list, draggable)` — attach the menu's gesture handlers to one cell
  list, with a flag deciding whether items in it can be picked up.

### Addressing an item

- `GetWeaponIndexByName(section) -> (group, index)` and `GetWeaponNameByIndex(group, index)` —
  the two directions of a **(group, index) coordinate** that identifies an item in the menu.
  That coordinate, not the section name, is what the network protocol and the preset machinery
  use, so both directions must exist and must agree.

### The economy

- `GetMoneyAmount` / `SetMoneyAmount` — the player's purse for this round.
- `IgnoreMoney(bool)` and `IgnoreMoneyAndRank(bool)` / `IsIgnoreMoneyAndRank` — server-side
  overrides that let a game type hand out free or unrestricted kit. Two separate switches
  because a game type may waive price without waiving rank.
- `SetRank` / `GetRank` — the rank every availability query is answered against; see
  [`Restrictions.h`](Restrictions.h.md).
- `CanBuyAllItems` — whether the current staged loadout is affordable and permitted in full.
- `CheckBuyAvailabilityInSlots` — the same question asked only of the equipped slots.

### Driving the menu from outside

- `SectionToSlot(group, index, real)` and `AddonToSlot(addon, slot, real)` — place an item or
  an attachment into a slot **without the player doing it**. The `real` flag distinguishes
  "make this the actual game state" from "show it only", which is how a server tells a client
  what it already owns.
- `ItemToSlot(section, addons)`, `ItemToBelt(section)`, `ItemToRuck(section, addons)` — the
  same by section, one per destination.
- `SetupPlayerItemsBegin` / `SetupPlayerItemsEnd` and
  `SetupDefaultItemsBegin` / `SetupDefaultItemsEnd` — **bracketing pairs**. Everything placed
  between a begin and its end is treated as one bulk setup: no sounds, no prices charged, no
  incremental revalidation. Two pairs because "what the player already has" and "what the game
  type gives you" are accumulated separately.
- `ResetItems` — clear the staged loadout.

### Presets

- `GetPreset(slot)` / `GetPresetCost(slot)` / `ClearPreset(slot)` / `TryUsePreset(slot)` —
  read, price, erase and recall. `TryUsePreset` is a *try*: a preset the player can no longer
  afford or is no longer ranked for is applied as far as it goes.

**Notes** — The width of this interface is the honest finding. A buy menu that can be driven
item-by-item from the network, that must distinguish real placements from displayed ones, and
that carries seven preset slots is a piece of *game logic wearing a screen*. A rebuild will
find most of this belongs behind a loadout model that the screen merely views, and the
interface is evidence that the original noticed and did not finish the separation.
