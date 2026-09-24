# src/xrUICore/ComboBox/UIComboBox.h

> Declares the drop-down — a closed text line that expands into a list box drawn above the rest of the UI, bound to a console variable's token set.

**Needs** — [`UIComboBox.cpp`](UIComboBox.cpp.md) · [`ListBox/UIListBox.h`](../ListBox/UIListBox.h.md) · [`EditBox/UIEditBox.h`](../EditBox/UIEditBox.h.md) · [`InteractiveBackground/UIInteractiveBackground.h`](../InteractiveBackground/UIInteractiveBackground.h.md) · [`Options/UIOptionsItem.h`](../Options/UIOptionsItem.h.md) · [`Windows/UIFrameWindow.h`](../Windows/UIFrameWindow.h.md)
**Used by** — [`UIMPPlayersAdm.cpp`](../../xrGame/ui/UIMPPlayersAdm.cpp.md) · [`UIMapList.cpp`](../../xrGame/ui/UIMapList.cpp.md) · [`UIComboBox.cpp`](UIComboBox.cpp.md) · [`UIXmlInitBase.cpp`](../XML/UIXmlInitBase.cpp.md)
**Tier floor** — T3.

## Purpose

Declares the surface implemented in [`UIComboBox.cpp`](UIComboBox.cpp.md).

Three structural decisions are visible in the header. The drop-down is a *composition*, not a
subclass: a four-state stretched-line background, a text static, a frame holding a list box —
all held by value and attached as children. It is a settings control, so its entries come from
a console variable's token table rather than from the screen. And it registers itself in the
device's render sequence, because an expanded list must be drawn after the rest of the UI or
it would be covered by later siblings.

## Exported units

- `CUIComboBox` — the drop-down.
- `InitComboBox(pos, width)` — build everything; the height is fixed at 20 virtual units
  closed, and the open height is the list's row height times the configured list length.
- `SetListLength(rows)` — how many rows the expanded list shows. Must be set before
  initialization and only once.
- `AddItem_(text, id)` — append an entry carrying an integer identifier.
- `ClearList` / `GetSize` / `GetText` / `GetTextOf(index)` / `SetText`.
- Selection: `SetItemIDX`, `SetItemToken(id)`, `GetSelectedIDX`, `SetSelectedIDX`,
  `SetNextItemSelected(forward, wrap)`, `CurrentID`.
- `disable_id(id)` / `enable_id(id)` — a filter applied when the entry list is rebuilt from
  the token table; a disabled identifier is simply not offered.
- `SetVertScroll` — whether the expanded list uses the fixed-thumb scroll bar.
- `SetTextColor` / `SetTextColorD` — the enabled and disabled text colours.
- The five settings-item operations.
- `OnRender` — the deferred draw of the expanded list.

**Notes** — the closed height constant of 20 units is the widget's own, not read from data.
Every combo box in every shipped screen is 20 units tall closed.
