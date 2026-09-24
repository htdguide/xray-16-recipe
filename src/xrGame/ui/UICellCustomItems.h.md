# src/xrGame/ui/UICellCustomItems.h

> Declares the three cell kinds — plain item, ammunition, weapon — and the buy menu's price
> overdraw.

**Needs** — [`UICellCustomItems.cpp`](UICellCustomItems.cpp.md) · [`UICellItem.h`](UICellItem.h.md)
**Used by** — [`UIActorMenuInventory.cpp`](UIActorMenuInventory.cpp.md) · [`UICellCustomItems.cpp`](UICellCustomItems.cpp.md) · [`UICellItemFactory.cpp`](UICellItemFactory.cpp.md) · [`UIMpTradeWnd_items.cpp`](UIMpTradeWnd_items.cpp.md) · [`UIMpTradeWnd_misc.cpp`](UIMpTradeWnd_misc.cpp.md) · [`UIMpTradeWnd_trade.cpp`](UIMpTradeWnd_trade.cpp.md)
**Tier floor** — T2: widget trees built from per-item configuration

## Purpose

Declares the surface implemented in [`UICellCustomItems.cpp`](UICellCustomItems.cpp.md).
These are the three products of [`create_cell_item`](UICellItemFactory.cpp.md), and the
differences between them are the whole content of the implementation twin.

## `SIconLayer`

One extra icon composited over an item's own icon, named by an item's configuration, with an
offset and a scale. The mechanism by which a modded item can have a layered icon without new
code.

## `CUIInventoryCellItem`

The base cell for any inventory item. Adds: the icon sub-rectangle and grid footprint taken
from the item's configuration; a list of icon layers; and the notion of a **helper item** — an
item present in the inventory only to back a stack, drawn dimmed and never draggable.

## `CUIAmmoCellItem`

Ammunition. Stacks with ammunition of the same section, and shows the **total number of
rounds** across the stack rather than the number of boxes.

## `CUIWeaponCellItem`

A weapon. Composites up to three attachment icons — silencer, scope, grenade launcher — over
the weapon icon at offsets the weapon's configuration supplies, and refuses to stack with a
weapon whose attachment set differs.

The three attachment kinds are a **closed enumeration**, frozen by the shipped weapon
configuration which names exactly these three attach points.

## `CBuyItemCustomDrawCell`

The buy menu's overdraw hook: prints a short string — a price — over a cell, in a caller-
supplied font. Holds at most 16 characters.
