# src/xrUICore/ListWnd/UIListItemEx.h

> Declares the list row that shows selection as a tinted background rather than as text highlighting.

**Needs** — [`UIListItemEx.cpp`](UIListItemEx.cpp.md) · [`UIListItem.h`](UIListItem.h.md)
**Used by** — [`UIListItemEx.cpp`](UIListItemEx.cpp.md)
**Tier floor** — T3.

## Purpose

Declares the surface implemented in [`UIListItemEx.cpp`](UIListItemEx.cpp.md).

The ordinary row has no persistent selected appearance — the list window draws a separate
highlight frame over whichever row is focused. This row carries its own: it starts fully
transparent and becomes a translucent tint when the list tells it that it is selected.

## Exported units

- `CUIListItemEx` — the self-tinting row.
- `SetSelectionColor(colour)` — the tint. Defaults to a warm brown at about four-fifths alpha.
- `SendMessage` — listens for the list's select and unselect messages.
