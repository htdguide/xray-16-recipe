# src/xrUICore/ListWnd/UIListItem.h

> Declares a row of the paged list window — a button carrying an index, a group identifier, an integer value and an opaque payload.

**Needs** — [`UIListItem.cpp`](UIListItem.cpp.md) · [`Buttons/UIButton.h`](../Buttons/UIButton.h.md)
**Used by** — [`UIListItemAdv.cpp`](../../xrGame/ui/UIListItemAdv.cpp.md) · [`UIListItemAdv.h`](../../xrGame/ui/UIListItemAdv.h.md) · [`UIListItem.cpp`](UIListItem.cpp.md) · [`UIListItemEx.cpp`](UIListItemEx.cpp.md) · [`UIListItemEx.h`](UIListItemEx.h.md) · [`UIListWnd.cpp`](UIListWnd.cpp.md) · [`UIListWnd.h`](UIListWnd.h.md) · [`UIListWnd_inline.h`](UIListWnd_inline.h.md)
**Tier floor** — T3.

## Purpose

Declares the surface implemented in [`UIListItem.cpp`](UIListItem.cpp.md).

The rows of the older list widget are *buttons*, not decorated panels: a row is pressed,
reports a click, and highlights under the pointer. Everything list-specific is metadata hung
on the button.

## Exported units

- `CUIListItem` — the row.
- `InitListItem(pos, size)` / `InitTexture(name)` — geometry, and a texture that additionally
  shifts the label right by the texture's width so the graphic and the label do not overlap.
- `GetIndex` / `SetIndex` — the row's position in the list. **Setting the index also sets the
  group identifier to the same value**, which is the default "each row is its own group".
- `GetGroupID` / `SetGroupID` — rows sharing a group highlight and select together. A group of
  -1 means "not part of any group" and is skipped by every group walk.
- `GetValue` / `SetValue` — a caller-assigned integer, searchable.
- `GetData` / `SetData` — an opaque caller pointer, searchable.
- `IsHighlightText` / `SetHighlightText` — whether the row's text draws highlighted. The
  default rule is "the pointer is over me"; the list window overrides the stored flag directly
  for whole groups.
- `MarkSelected` — a hook for subclasses; the base does nothing.

**Notes** — rows mark themselves auto-delete at construction, so the list window owns them by
attaching them and never releases them explicitly.
