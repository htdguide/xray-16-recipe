# src/xrGame/ui/UICellItemFactory.h

> Declares the one function that turns an inventory item into the cell widget that represents it.

**Needs** — [`UICellItemFactory.cpp`](UICellItemFactory.cpp.md)
**Used by** — [`UIActorMenuDeadBodySearch.cpp`](UIActorMenuDeadBodySearch.cpp.md) · [`UIActorMenuInventory.cpp`](UIActorMenuInventory.cpp.md) · [`UIActorMenuTrade.cpp`](UIActorMenuTrade.cpp.md) · [`UIDragDropReferenceList.cpp`](UIDragDropReferenceList.cpp.md)
**Tier floor** — T3: one dispatch on item kind

## Purpose

Declares the surface implemented in [`UICellItemFactory.cpp`](UICellItemFactory.cpp.md).

## `create_cell_item`

**Contract** — Takes an inventory item, returns a newly built cell widget of the subclass
appropriate to that item's kind. Ownership passes to the caller, which normally hands it
straight to a cell list.
