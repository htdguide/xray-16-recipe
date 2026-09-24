# src/xrUICore/ListWnd/UIListWnd.h

> Declares the row-paged list — a fixed-height-row list that shows a whole number of rows, scrolls by whole rows, and tracks a focused row and a selected row separately.

**Needs** — [`UIListWnd.cpp`](UIListWnd.cpp.md) · [`UIListWnd_inline.h`](UIListWnd_inline.h.md) · [`UIListItem.h`](UIListItem.h.md) · [`Windows/UIWindow.h`](../Windows/UIWindow.h.md) · [`ScrollBar/UIScrollBar.h`](../ScrollBar/UIScrollBar.h.md) · [`Windows/UIFrameLineWnd.h`](../Windows/UIFrameLineWnd.h.md)
**Used by** — [`UIScriptWnd_script.cpp`](../../xrGame/ui/UIScriptWnd_script.cpp.md) · [`UIListWnd.cpp`](UIListWnd.cpp.md) · [`UIListWnd_inline.h`](UIListWnd_inline.h.md) · [`UIXmlInitBase.cpp`](../XML/UIXmlInitBase.cpp.md)
**Tier floor** — T3.

## Purpose

Declares the surface implemented in [`UIListWnd.cpp`](UIListWnd.cpp.md) and
[`UIListWnd_inline.h`](UIListWnd_inline.h.md).

This is the older of the two list widgets. Its defining decision is that scrolling is
*discrete*: the scroll position is a row index, the visible window is a whole number of rows,
and a row is either fully visible or hidden. That is why it can place rows by index
arithmetic and never needs clipping. The newer [`CUIListBox`](../ListBox/UIListBox.h.md)
scrolls by pixels instead.

Its second decision is two independent cursors: a *focused* row — the one under the pointer or
force-set by a caller — and a *selected* row — the one last clicked. Both can be drawn with a
highlight frame, independently.

## Exported units

- `CUIListWnd` — the list.
- `InitListWnd(pos, size, row_height)` — creates and places the scroll bar to the left of the
  right edge, computes how many rows fit, and sizes rows to the remaining width.
- `AddItem<Element>(text, shift, data, value, insert_before)` and `AddItem<Element>(item,
  insert_before)` — see [`UIListWnd_inline.h`](UIListWnd_inline.h.md).
- `RemoveItem(index)` / `RemoveAll` / `GetItem(index)` / `GetItemsCount` / `GetItemPos(item)`.
- `FindItem(data)` / `FindItemWithValue(value)` — linear searches returning an index or -1.
- Scrolling: `ScrollToBegin`, `ScrollToEnd`, `ScrollToPos(index, centre_ratio)`,
  `GetListPosition`, `EnableScrollBar`, `IsScrollBarEnabled`, `UpdateScrollBar`,
  `SetAlwaysShowScroll`, `EnableAlwaysShowScroll`.
- Cursors: `SetFocusedItem`, `GetFocusedItem`, `GetSelectedItem`, `SetSelectedItem`,
  `ShowSelectedItem`, `ResetFocusCapture`, `EnableActiveBackground`.
- Appearance and geometry: `SetItemWidth` / `SetItemHeight` / `SetWidth` / `SetHeight`,
  `SetTextColor`, `SetScrollBarProfile`, `SetVertFlip`.
- `ActivateList(on)` — whether rows accept input at all; a deactivated list still draws.

**Notes** — `SetVertFlip` stacks rows from the bottom of the widget upward instead of from the
top down. It is used by the in-game message log, which grows upward.
