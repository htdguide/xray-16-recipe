# src/xrUICore/PropertiesBox/UIPropertiesBox.h

> Declares the pop-up context menu implemented in [`UIPropertiesBox.cpp`](UIPropertiesBox.cpp.md).

**Needs** — [`UIPropertiesBox.cpp`](UIPropertiesBox.cpp.md) · [`Windows/UIFrameWindow.h`](../Windows/UIFrameWindow.h.md) · [`ListBox/UIListBox.h`](../ListBox/UIListBox.h.md) · [`Callbacks/UIWndCallback.h`](../Callbacks/UIWndCallback.h.md) · [Seam: Script binding layer](../../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer)
**Used by** — [`UIActorMenuInitialize.cpp`](../../xrGame/ui/UIActorMenuInitialize.cpp.md) · [`UIActorMenuInventory.cpp`](../../xrGame/ui/UIActorMenuInventory.cpp.md) · [`UIActorMenu_action.cpp`](../../xrGame/ui/UIActorMenu_action.cpp.md) · [`UIDemoPlayControl.cpp`](../../xrGame/ui/UIDemoPlayControl.cpp.md) · [`UIMapWnd.cpp`](../../xrGame/ui/UIMapWnd.cpp.md) · [`UIScriptWnd_script.cpp`](../../xrGame/ui/UIScriptWnd_script.cpp.md) · [`UIPropertiesBox.cpp`](UIPropertiesBox.cpp.md) · [`ui_export_script.cpp`](../ui_export_script.cpp.md)
**Tier floor** — T3: a declaration.

## Purpose

Declares the type implemented in [`UIPropertiesBox.cpp`](UIPropertiesBox.cpp.md). Two shape
decisions are visible only here.

**A properties box is a framed window wrapping a list box**, not a list box with a frame.
The frame owns the nine-slice art and the placement logic; the list owns the items,
selection and keyboard navigation. Nothing else is added.

**Submenus are a two-element chain, not a tree.** A box is constructed holding an optional
pointer to the one box that may open beside it, and that child is told who its parent is —
so the same relationship is recorded twice, in both directions, which the source itself
flags as a hazard. A rebuild should hold one edge and derive the other. The chain is
depth-limited by construction: a box has at most one child submenu and no recursion.

## Exported units

- `CUIPropertiesBox` — the menu.
- `InitPropertiesBox(position, size)` — build from the shipped menu layout.
- `AddItem(text, payload, tag)` / `AddItem_script(text)` — append an entry; the payload and
  the tag are the caller's identity handles for the entry.
- `RemoveItemByTAG` / `RemoveAll` / `GetItemsCount`.
- `Show(bounding_rect, anchor_point)` — open, placed by the four-quadrant rule in the
  implementation. Note this *shadows* the window's own one-argument show: opening a
  properties box always requires somewhere to open it.
- `Hide` — close, releasing every capture it took.
- `GetClickedItem` — the selected entry, valid when the click notification fires.
- `AutoUpdateSize` — shrink-wrap the box around its entries.
- `ShowSubMenu` — open the child box beside the entry that is currently selected.
- `OnItemReceivedFocus` — close the child box when focus moves to a different entry.

**Notes** — the destructor asserts that the child submenu is already closed. That is not
tidiness: a closing child hides its parent, so destroying a parent under an open child would
drive a hide through freed memory. A rebuild expresses the same constraint as "a box may not
be destroyed while its submenu is open", however its ownership model states that.
