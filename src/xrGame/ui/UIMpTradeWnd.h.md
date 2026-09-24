# src/xrGame/ui/UIMpTradeWnd.h

> Declares the multiplayer buy screen: the browsable store, the player's ten destination lists,
> the item record that tracks what was bought or sold, and five saved loadouts.

**Needs** — [`UIMpTradeWnd.cpp`](UIMpTradeWnd.cpp.md) · [`UIMpTradeWnd_init.cpp`](UIMpTradeWnd_init.cpp.md) · [`UIMpTradeWnd_items.cpp`](UIMpTradeWnd_items.cpp.md) · [`UIMpTradeWnd_trade.cpp`](UIMpTradeWnd_trade.cpp.md) · [`UIMpTradeWnd_misc.cpp`](UIMpTradeWnd_misc.cpp.md) · [`UIMpTradeWnd_wpn.cpp`](UIMpTradeWnd_wpn.cpp.md) · [`UIMpItemsStoreWnd.h`](UIMpItemsStoreWnd.h.md) · [`UIBuyWndBase.h`](UIBuyWndBase.h.md) · [`UIBuyWndShared.h`](UIBuyWndShared.h.md) · [`UICellItem.h`](UICellItem.h.md) · [`Restrictions.h`](Restrictions.h.md) · [`xrUICore/Callbacks/UIWndCallback.h`](../../xrUICore/Callbacks/UIWndCallback.h.md)
**Used by** — [`UIGameCTA.cpp`](../UIGameCTA.cpp.md) · [`game_cl_deathmatch_buywnd.cpp`](../game_cl_deathmatch_buywnd.cpp.md) · [`UIBuyWndShared.cpp`](UIBuyWndShared.cpp.md) · [`UIMpTradeWnd.cpp`](UIMpTradeWnd.cpp.md) · [`UIMpTradeWnd_init.cpp`](UIMpTradeWnd_init.cpp.md) · [`UIMpTradeWnd_items.cpp`](UIMpTradeWnd_items.cpp.md) · [`UIMpTradeWnd_misc.cpp`](UIMpTradeWnd_misc.cpp.md) · [`UIMpTradeWnd_trade.cpp`](UIMpTradeWnd_trade.cpp.md) · [`UIMpTradeWnd_wpn.cpp`](UIMpTradeWnd_wpn.cpp.md)
**Tier floor** — T3.

## Purpose

Declares the surface implemented across five files, and — because it is where they are
declared — is the substance holder for three vocabularies the implementation only uses.
The screen itself is described in [`UIMpTradeWnd.cpp`](UIMpTradeWnd.cpp.md); construction in
[`_init`](UIMpTradeWnd_init.cpp.md); the item table and loadouts in
[`_items`](UIMpTradeWnd_items.cpp.md); buying and selling in
[`_trade`](UIMpTradeWnd_trade.cpp.md); input, money and drag routing in
[`_misc`](UIMpTradeWnd_misc.cpp.md); weapon attachments in [`_wpn`](UIMpTradeWnd_wpn.cpp.md).

## `SBuyItemInfo` — the item record

**Contract** — one record per item the screen knows about, in any state. It pairs an item
section with the cell widget that represents it, and carries a state that is *not* freely
assignable — see [`_items`](UIMpTradeWnd_items.cpp.md), where the transition table lives.
Destroying the record destroys both the widget and the game-object instance behind it.

```text
RECORD BuyItem
  name_sect  : text
  cell_item  : CellItem        # its widget; the widget's payload is a live inventory item
  state      : ItemState

ENUM ItemState
  undefined   # just created, not yet placed
  bought      # taken from the shop this session
  sold        # given back this session
  own         # the player already had it when the screen opened
  shop        # on the shelf
```

## The list vocabulary

**Contract** — ten drag-and-drop lists in a fixed order, which is *also* the order of the
layout document's list elements and the index space the price table's slot lookup returns.

```text
ENUM ListId
  pistol, pistol_ammo, rifle, rifle_ammo, outfit, medkit, grenade, others,
  player_bag,          # everything not in a slot
  shop                 # the shelf; the only list not owned by the player
```

**Invariants** — the four *slot* lists — pistol, rifle, outfit — hold at most one item each;
the ammunition lists sit immediately after their weapon in the enumeration, and the code
derives one from the other by index arithmetic, so the adjacency is load-bearing.

## The remaining vocabularies

```text
ENUM ListKind      shop, own_bag, own_slot          # what a list means for drag rules
ENUM AddonKind     scope = 1, grenade_launcher = 2, silencer = 4   # a bit mask on a weapon
FLAGS BuyFlags     check_money, check_rank, check_group_count,
                   mark_as_own, ignore_team
                   normal = check_money | check_rank | check_group_count
```

**Notes** — the attachment values are the same bits the weapon's own attachment state uses, so
a weapon's attachments and this screen's attachment requests are one representation. That is
frozen by the weapon layer, not by this screen.

## Exported units

- **The buy screen** — a modal dialog implementing the shared buy-screen interface, so that
  the game layer can drive it without knowing which of the two buy screens it has.
- `Init` — build from the buy layout document against a named item section and price section.
- `Show`, `Update`, `OnKeyboardAction`, `SendMessage` — the lifecycle.
- The money and rank surface: `GetMoneyAmount`, `SetMoneyAmount`, `SetRank`, `GetRank`,
  `IgnoreMoney`, `IgnoreMoneyAndRank`, `IsIgnoreMoneyAndRank`.
- The loadout surface: `GetPreset`, `GetPresetCost`, `ClearPreset`, `TryUsePreset`.
- The population surface the game layer calls to describe what the player already owns:
  `SetupPlayerItemsBegin`/`End`, `SetupDefaultItemsBegin`/`End`, `ItemToSlot`, `ItemToBelt`,
  `ItemToRuck`, `ResetItems`.
- `GetWeaponIndexByName` / `GetWeaponNameByIndex` — the compact index the network protocol
  names an item by.
- `HasItemInGroup`, `GetItemMngr`, `BindDragDropListEvents`.
- **`CUICellItemTradeMenuDraw`** — the per-cell overlay that tints a shelf item by whether it
  can be afforded and draws its keyboard accelerator.
