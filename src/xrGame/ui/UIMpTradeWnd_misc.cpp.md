# src/xrGame/ui/UIMpTradeWnd_misc.cpp

> Where an item is allowed to go, what a drag or a click means once it gets there, the money
> readouts, and the rank badge.

**Needs** — [`UIMpTradeWnd.h`](UIMpTradeWnd.h.md) · [`UIMpItemsStoreWnd.h`](UIMpItemsStoreWnd.h.md) · [`UICellItem.h`](UICellItem.h.md) · [`UICellCustomItems.h`](UICellCustomItems.h.md) · [`UIDragDropListEx.h`](UIDragDropListEx.h.md) · [`UIItemInfo.h`](UIItemInfo.h.md) · [`inventory_item.h`](../inventory_item.h.md) · [`Restrictions.h`](Restrictions.h.md) · [`xrUICore/TabControl/UITabControl.h`](../../xrUICore/TabControl/UITabControl.h.md)
**Used by** — [`UIMpTradeWnd.h`](UIMpTradeWnd.h.md)
**Tier floor** — T3.

## Purpose

The screen's interaction rules. The placement rule in particular is the piece a rebuild is most
likely to get wrong, because it is a *fallback chain* rather than a table.

## `GetMatchedListForItem` — the placement rule

**Contract** — answer which list an item section belongs in, given what the player currently
holds.

```text
FUNCTION matched_list(section) -> List
  list <- price_table.destination_list(section)       # authored, per item
  IF list IS an ammunition list THEN
    weapon <- the item in the adjacent weapon slot, if any
    IF there is none, or it does not take this ammunition THEN RETURN bag
  IF list IS one of (pistol, rifle, outfit) AND that list is occupied THEN RETURN bag
  RETURN list
```

**Notes** — the **bag is the universal fallback**, and reaching it is not a failure: it is where
a second rifle, or ammunition for a rifle you no longer carry, correctly lives. The ammunition
case is why the enumeration puts each ammunition list immediately after its weapon — the
adjacency is read as `list - 1`. A rebuild should name the pairing instead of relying on the
order, and must keep the *rule*: ammunition is only ammunition while you hold the weapon.

## Drag and drop

**Contract** — five handlers, bound to every list at construction.

- **Drop** — nothing happens when the item did not move. Dragging *out of* the shelf buys.
  Dragging *into* the shelf sells. Dragging into the bag always succeeds and re-derives the
  source list's ammunition. Dragging into a slot succeeds **only if that slot is the one the
  placement rule chose** for the item — so a rifle cannot be dropped into the pistol slot, and a
  second rifle cannot be dropped onto the first.
- **Double click** — on the shelf, buy. In one of the four *refillable* lists — the two
  ammunition lists, medical, grenades — buy another one. Anywhere else on the player's side,
  sell.
- **Left click** — in a refillable list only, buy another of that section. Elsewhere, decline.
- **Right click** — in a refillable list only: delete that list's automatic items, find a real
  bought or owned copy of the section, sell it, and rebuild. Elsewhere, decline.
- **Select** — show the item in the detail panel. Never consumes the event.
- **Start drag** — always declines, so dragging is allowed everywhere.

**Notes** — **the four refillable lists behave completely differently from the rest of the
screen**: left click adds one, right click removes one, and double click adds. That is a
quantity editor grafted onto a drag-and-drop grid, and it exists because those lists hold many
copies of a few sections while everything else holds one of each. The right-click path has to
delete and rebuild the automatic items around the sale because selling a section that is
*also* present as an automatic item would otherwise sell the free copy.

## Keyboard

**Contract** — while browsing below the root, keys go to the shop panel first — that is how the
digit accelerators pick a shelf item — and the tab strip's own accelerators are suppressed for
the duration so a digit does not also change tabs. Three letter keys buy one round of pistol
ammunition, rifle ammunition and launcher ammunition respectively. A development build binds two
keypad keys to stepping the player's rank.

**Notes** — the three letter keys are the only way to reach the three attachment-panel buttons
that were removed from the layout; see [`_init`](UIMpTradeWnd_init.cpp.md). They are raw
scancodes, not bound actions, so they are not rebindable and not localised — a rebuild should
route them through the binding table.

## The money readouts

**Contract** — refreshed every frame. The player's money is shown as a number, or as dashes when
money is being ignored. Each of the five loadout buttons shows its loadout's cost, tinted by
whether the player can afford it, and is enabled only when it is both affordable and non-empty.
Once every thirty frames the current selection is stored into a scratch loadout and its cost
shown as "what you are about to spend".

**Notes** — the current cost is computed by *storing a loadout and pricing it*, which is a
surprisingly heavy way to total a list and is why it runs at one frame in thirty. It is also
correct in a way a naive sum is not: the loadout collapses duplicates and prices attachments,
so it counts an attached scope once and at its own price.

A loadout's cost is the sum over entries of (item price + its attachments' prices) times the
count — so attachments are charged per copy, which matters for an entry recording two identical
scoped rifles.

## `SetCurrentItem` and the rank badges

**Contract** — selecting an item fills the detail panel with its name and its price at the
player's rank, and shows a rank badge texture for the *item's required rank*, named by team
colour and rank number. Setting the player's rank does the same for the player's own badge and
tells the restriction table. When money and rank are being ignored, the player's rank is forced
to the maximum, money to its largest value, and the player's badge is hidden.

**Notes** — the badge for an item the *other* team sells is meant to use the other team's
colour, and the expression that picks it reduces to zero rather than flipping the index, so it
always shows the first team's colour. The intent is clear from the shape of the code; the
correct behaviour is not recoverable from it, only guessable.

## The compact item index

**Contract** — `GetWeaponIndexByName` and `GetWeaponNameByIndex` translate between an item
section and its position in the price table, which is how the network protocol names an item in
a single byte. The group number is always zero — the grouping the interface allows for is
unused by this screen.

**Notes** — this is the only part of the file with a frozen external contract: the index is the
price table's order, and both ends of the network message must agree on it. A rebuild may
reorder the price table only if it reorders both.

## The rest

**Contract** — `GetItemPrice` is the price table at the current rank. `SetInfoString` and
`SetMoneyChangeString` write the two message lines and restart their fade animations, the money
line signed and tinted by direction and suffixed with the localized currency name.
`ResetItems` returns to the arrival state, empties the player's lists, and returns the store to
its root. `CanBuyAllItems` and `CheckBuyAvailabilityInSlots` answer yes unconditionally, and
`IgnoreMoney`, `AddonToSlot` and `SectionToSlot` do nothing: all five satisfy the shared
buy-screen interface that the *other* buy screen implements meaningfully.
