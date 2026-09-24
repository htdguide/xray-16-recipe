# src/xrGame/ui/UIMpItemsStoreWnd.h

> Declares the store's category tree: an authored hierarchy of named levels whose leaves list
> the item sections on sale, with a cursor for the level currently being browsed.

**Needs** — [`UIMpItemsStoreWnd.cpp`](UIMpItemsStoreWnd.cpp.md) · [`UIBuyWndShared.h`](UIBuyWndShared.h.md) · [`UITabButtonMP.h`](UITabButtonMP.h.md) · [`Common/object_interfaces.h`](../../Common/object_interfaces.h.md)
**Used by** — [`UIMpItemsStoreWnd.cpp`](UIMpItemsStoreWnd.cpp.md) · [`UIMpTradeWnd.cpp`](UIMpTradeWnd.cpp.md) · [`UIMpTradeWnd.h`](UIMpTradeWnd.h.md) · [`UIMpTradeWnd_init.cpp`](UIMpTradeWnd_init.cpp.md) · [`UIMpTradeWnd_items.cpp`](UIMpTradeWnd_items.cpp.md) · [`UIMpTradeWnd_misc.cpp`](UIMpTradeWnd_misc.cpp.md) · [`UIMpTradeWnd_trade.cpp`](UIMpTradeWnd_trade.cpp.md) · [`UIMpTradeWnd_wpn.cpp`](UIMpTradeWnd_wpn.cpp.md)
**Tier floor** — T3.

## Purpose

Declares the surface implemented in [`UIMpItemsStoreWnd.cpp`](UIMpItemsStoreWnd.cpp.md).
Despite the file name there is no window here: the type is a *tree*, and the buy screen draws
it. The name is historical and a rebuild should rename it.

## Exported units

- **`CStoreHierarchy`** — the tree plus a cursor into it.
- **`CStoreHierarchy::item`** — one node: a name, an optional tab button, child nodes, and —
  at a leaf — the list of item sections in that category.
- `Init` — build the tree's shape from a layout document subtree.
- `InitItemsInGroup` — fill every leaf's item list from a configuration section, and record
  which team that section belongs to.
- `Reset`, `MoveUp`, `MoveDown`, `CurrentLevel`, `CurrentIsRoot` — the browsing cursor.
- `FindItem` — which leaf sells a given item section, or nothing.
- `TeamIdx` — the team the loaded store belongs to.
- On a node: `HasItem`, `GetItemIdx`, `ChildCount`, `Child`, `ChildAtIdx`, `HasSubLevels`.
