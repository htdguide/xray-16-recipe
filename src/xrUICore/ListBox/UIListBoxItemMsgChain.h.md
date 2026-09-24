# src/xrUICore/ListBox/UIListBoxItemMsgChain.h

> Declares a list row that performs its selection and then lets the press keep travelling to widgets behind it.

**Needs** — [`UIListBoxItemMsgChain.cpp`](UIListBoxItemMsgChain.cpp.md) · [`UIListBoxItem.h`](UIListBoxItem.h.md)
**Used by** — [`UIListBoxItemMsgChain.cpp`](UIListBoxItemMsgChain.cpp.md)
**Tier floor** — T3.

## Purpose

Declares the type implemented in [`UIListBoxItemMsgChain.cpp`](UIListBoxItemMsgChain.cpp.md).

One behaviour, one method. The ordinary row consumes a press; this one does not, so the press
continues up the dispatch and reaches the row's ancestors. It exists for lists whose rows are
decoration over a larger clickable area.

## Exported units

- `CUIListBoxItemMsgChain` — the non-consuming row.
