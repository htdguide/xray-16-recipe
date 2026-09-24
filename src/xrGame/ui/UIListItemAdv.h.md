# src/xrGame/ui/UIListItemAdv.h

> Declares the multi-column list row. Excluded from the build.

**Needs** — [`UIListItemAdv.cpp`](UIListItemAdv.cpp.md) · [`UIListItem.h`](../../xrUICore/ListWnd/UIListItem.h.md)
**Used by** — [`UIListItemAdv.cpp`](UIListItemAdv.cpp.md)
**Tier floor** — T3.

## Purpose

Declares the surface implemented in [`UIListItemAdv.cpp`](UIListItemAdv.cpp.md). Both files
are commented out of the build.

## Exported units

- **`CUIListItemAdv`**
  - `AddField(text, width)` — append a text column.
  - `AddWindow(widget)` — append an arbitrary widget, vertically centred.
  - `SetTextColor` — recolour the row and every text column together.
  - `GetNextLeftPos` — the running right edge, derived from the children.
