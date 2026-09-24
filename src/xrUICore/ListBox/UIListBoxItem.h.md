# src/xrUICore/ListBox/UIListBoxItem.h

> Declares one row of a list box — a stretched-line background drawn only when selected, over a left-to-right sequence of text and icon fields.

**Needs** — [`UIListBoxItem.cpp`](UIListBoxItem.cpp.md) · [`Windows/UIFrameLineWnd.h`](../Windows/UIFrameLineWnd.h.md) · [`uiabstract.h`](../uiabstract.h.md) · [`Static/UIStatic.h`](../Static/UIStatic.h.md)
**Used by** — [`UIDemoPlayControl.cpp`](../../xrGame/ui/UIDemoPlayControl.cpp.md) · [`UIListItemServer.cpp`](../../xrGame/ui/UIListItemServer.cpp.md) · [`UIListItemServer.h`](../../xrGame/ui/UIListItemServer.h.md) · [`UIComboBox.cpp`](../ComboBox/UIComboBox.cpp.md) · [`UIListBox.cpp`](UIListBox.cpp.md) · [`UIListBox.h`](UIListBox.h.md) · [`UIListBoxItem.cpp`](UIListBoxItem.cpp.md) · [`UIListBoxItemMsgChain.cpp`](UIListBoxItemMsgChain.cpp.md) · [`UIListBoxItemMsgChain.h`](UIListBoxItemMsgChain.h.md) · [`UIPropertiesBox.cpp`](../PropertiesBox/UIPropertiesBox.cpp.md)
**Tier floor** — T3.

## Purpose

Declares the surface implemented in [`UIListBoxItem.cpp`](UIListBoxItem.cpp.md).

A row is a *multi-column* thing even though most rows have one column: fields are appended
left to right, each one placed where the previous one ended. The first field is created by the
constructor and is the row's "the text"; everything else is opt-in.

## Exported units

- `CUIListBoxItem` — the row. Selectable, so it carries a selected flag the list box sets.
- `AddTextField(text, width)` / `AddIconField(width)` — append a field at the current right
  edge, full row height.
- `GetTextItem` — the first field.
- `SetText` / `GetText` / `SetTextColor` / `GetTextColor` / `SetFont` / `GetFont` — forward to
  the first field.
- `SetTAG` / `GetTAG` — a caller-assigned identifier, the payload of every message the row
  sends. Starts as the all-ones sentinel.
- `SetData` / `GetData` — an opaque caller pointer, not sent with messages.
- `InitDefault` — load the built-in row-highlight texture.

**Notes** — the tag is sent *by address*, so a listener reads it out of the row and must not
retain the pointer past the handler.
