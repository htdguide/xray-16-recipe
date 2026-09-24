# src/xrGame/ui/UIDragDropListEx.h

> Declares the cell board, the cell record, and the hook surface a screen installs to give the board meaning.

**Needs** — [`UIDragDropListEx.cpp`](UIDragDropListEx.cpp.md) · [`UICellItem.h`](UICellItem.h.md) · [`xrUICore/Windows/UIWindow.h`](../../xrUICore/Windows/UIWindow.h.md) · [`xrUICore/Callbacks/UIWndCallback.h`](../../xrUICore/Callbacks/UIWndCallback.h.md)
**Used by** — [`UIActorMenu.cpp`](UIActorMenu.cpp.md) · [`UIActorMenuDeadBodySearch.cpp`](UIActorMenuDeadBodySearch.cpp.md) · [`UIActorMenuInitialize.cpp`](UIActorMenuInitialize.cpp.md) · [`UIActorMenuInventory.cpp`](UIActorMenuInventory.cpp.md) · [`UIActorMenuTrade.cpp`](UIActorMenuTrade.cpp.md) · [`UIActorMenuUpgrade.cpp`](UIActorMenuUpgrade.cpp.md) · [`UIActorMenu_action.cpp`](UIActorMenu_action.cpp.md) · [`UICellCustomItems.cpp`](UICellCustomItems.cpp.md) · [`UICellItem.cpp`](UICellItem.cpp.md) · [`UIDragDropListEx.cpp`](UIDragDropListEx.cpp.md) · [`UIDragDropReferenceList.cpp`](UIDragDropReferenceList.cpp.md) · [`UIDragDropReferenceList.h`](UIDragDropReferenceList.h.md) · [`UIMpTradeWnd.cpp`](UIMpTradeWnd.cpp.md) · [`UIMpTradeWnd_init.cpp`](UIMpTradeWnd_init.cpp.md) · _and 6 more_
**Tier floor** — T2.

## Purpose

Declares the surface implemented in [`UIDragDropListEx.cpp`](UIDragDropListEx.cpp.md). Two
types live here and one of them, the cell, is a record rather than a widget.

## `CUICell`

**Contract** — one board position: the item occupying it, and whether this is that item's
top-left cell. Clearing a cell also severs the item's back-link to its owning list, so a
cleared cell cannot leave a dangling ownership claim. **Two cells compare equal when they
hold the same item** — the comparison is by occupant, not by position, and the draw path
relies on it to collapse a multi-cell item to one entry.

## `CUIDragDropListEx`

**Contract** — the viewport widget. Its surface falls into five groups:

- **Shape** — cells capacity (current, authored-start, maximum), cell size, cell spacing,
  virtual-cell alignment; `CalculateCapacity` turns a desired slot count into a shape
  consistent with the authored markup.
- **Mode flags** — group similar, auto grow, custom placement, vertical placement, always
  show scroll, virtual cells. Six independent bits; the combinations are what make one widget
  serve backpack, belt, slot and trade pane.
- **Items** — place (automatically, at a point, at a cell), test whether an item can be
  placed, remove, count, index, ownership test, clear, compact, and the armament-highlight
  reset.
- **Decoration** — the highlighter, the blocker and the condition indicator, each attached
  with a per-cell spacing and adopted by the list.
- **Hooks** — the ten veto callbacks, public fields the screen assigns directly.

`m_drag_item` is **static**: one drag in flight for the whole process. See the
implementation twin.

## `CUICellContainer`

**Contract** — the board itself: the cell vector, its geometry, the placement search, the
stacking rule, and the batched grid draw. Almost its entire surface is reachable only by the
list and by the reference-list variant; a rebuild may fold it into the list, at the cost of
losing the one honest distinction it draws — the list is the *viewport and the policy*, the
container is the *board*.

Publicly it offers only cell picking, cell validity and cell access, which is what screens
need to reason about positions without owning the board.

**Notes** — the board's cell vector is row-major and indexed arithmetically, so a rebuild
must keep the row stride equal to the column capacity or every lookup shifts. `Shrink` is
declared and does nothing: capacity only grows.
