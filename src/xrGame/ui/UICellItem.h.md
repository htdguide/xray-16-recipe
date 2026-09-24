# src/xrGame/ui/UICellItem.h

> Declares the item-in-a-grid widget, the floating widget that represents it while it is being
> dragged, and the two hooks that let a screen overdraw either one.

**Needs** — [`UICellItem.cpp`](UICellItem.cpp.md) · [`../../xrUICore/Static/UIStatic.h`](../../xrUICore/Static/UIStatic.h.md) · [`UIDialogWnd.h`](UIDialogWnd.h.md)
**Used by** — [`UIGameCustom.cpp`](../UIGameCustom.cpp.md) · [`UIActorMenu.cpp`](UIActorMenu.cpp.md) · [`UIActorMenuUpgrade.cpp`](UIActorMenuUpgrade.cpp.md) · [`UIActorMenu_action.cpp`](UIActorMenu_action.cpp.md) · [`UIActorMenu_script.cpp`](UIActorMenu_script.cpp.md) · [`UICellCustomItems.cpp`](UICellCustomItems.cpp.md) · [`UICellCustomItems.h`](UICellCustomItems.h.md) · [`UICellItem.cpp`](UICellItem.cpp.md) · [`UICellItemFactory.cpp`](UICellItemFactory.cpp.md) · [`UIDragDropListEx.cpp`](UIDragDropListEx.cpp.md) · [`UIDragDropListEx.h`](UIDragDropListEx.h.md) · [`UIDragDropReferenceList.cpp`](UIDragDropReferenceList.cpp.md) · [`UIItemInfo.cpp`](UIItemInfo.cpp.md) · [`UIMpTradeWnd.cpp`](UIMpTradeWnd.cpp.md) · _and 6 more_
**Tier floor** — T2: widget tree with a per-frame draw and a process-wide drag anchor

## Purpose

Declares the surface implemented in [`UICellItem.cpp`](UICellItem.cpp.md), plus two abstract
hooks that *are* substance and are written in full below.

## `ICustomDrawCellItem`

**Contract** — The interface a screen implements to draw something extra over a cell after the
cell has drawn itself: `OnDraw(cell)` is called at the end of the cell's own draw, with the
cell available for its absolute rectangle. Exactly one such hook may be installed per cell,
and installing a second **destroys the first** — the cell owns its hook.

**Invariants** — The hook is called during draw, so it may emit geometry and must not mutate
the widget tree. It sees the cell after its children, so anything it draws is on top of
everything the cell drew.

**Notes** — This is how the multiplayer buy menu writes a price over an item icon without the
cell widget knowing what a price is; see
[`UICellCustomItems.cpp`](UICellCustomItems.cpp.md).

## `ICustomDrawDragItem`

**Contract** — The same hook for the floating drag widget: `OnDraw(drag_item)` is called after
the drag widget draws, with the drag widget available for its current position and size. Same
single-owner, destroy-on-replace rule.

**Notes** — This is how the inventory screen puts a trash-can badge next to the cursor when a
drag hovers the trash target: the badge is not part of the item, it is a comment on the drop.

## `CUICellItem`

One item occupying a rectangle of grid cells. A picture widget with a stack count, an upgrade
marker, a condition bar, an accelerator key, and a list of *children* — the other items merged
into this one's stack. Contracts in [`UICellItem.cpp`](UICellItem.cpp.md).

Notable members that are part of the contract rather than implementation detail:

- `m_pData` — the untyped payload. A cell does not know what an item is; the screen casts it.
- `m_mouse_selected_item` — **a single process-wide anchor** recording which cell the current
  press began on. See the implementation twin: this is load-bearing and is the reason a drag
  cannot start on a cell the press did not begin on.
- `m_grid_size` — the item's footprint in grid cells, taken from the item's configuration.
- `m_selected`, `m_select_armament`, `m_select_equipped`, `m_cur_mark` — four independent
  highlight flags, each set by a different rule; the drawing combines them.
- `m_drawn_frame` — the last frame this cell drew, used by the list to tell visible cells from
  ones scrolled out.
- `m_b_destroy_childs` — whether destruction also destroys the merged children. False when the
  children have been handed to somebody else.

## `CUIDragItem`

The floating widget that follows the cursor during a drag. Unlike every other widget in the
UI it registers itself directly with the frame loop's render and update lists at a priority
that puts it **after** all other UI, which is how it draws over every screen including the
one that owns it. Contracts in [`UICellItem.cpp`](UICellItem.cpp.md).
