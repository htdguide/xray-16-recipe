# src/xrUICore/ListBox/UIListBox.h

> Declares the scrolling list of selectable rows — a scroll view that knows its children are list items and can find, select and reorder them.

**Needs** — [`UIListBox.cpp`](UIListBox.cpp.md) · [`UIListBoxItem.h`](UIListBoxItem.h.md) · [`ScrollView/UIScrollView.h`](../ScrollView/UIScrollView.h.md)
**Used by** — [`ServerList.h`](../../xrGame/ui/ServerList.h.md) · [`UIChangeMap.cpp`](../../xrGame/ui/UIChangeMap.cpp.md) · [`UIKickPlayer.cpp`](../../xrGame/ui/UIKickPlayer.cpp.md) · [`UIMPChangeMapAdm.cpp`](../../xrGame/ui/UIMPChangeMapAdm.cpp.md) · [`UIMPPlayersAdm.cpp`](../../xrGame/ui/UIMPPlayersAdm.cpp.md) · [`UIMapList.cpp`](../../xrGame/ui/UIMapList.cpp.md) · [`UIVote.cpp`](../../xrGame/ui/UIVote.cpp.md) · [`UIComboBox.cpp`](../ComboBox/UIComboBox.cpp.md) · [`UIComboBox.h`](../ComboBox/UIComboBox.h.md) · [`UIListBox.cpp`](UIListBox.cpp.md) · [`UIPropertiesBox.cpp`](../PropertiesBox/UIPropertiesBox.cpp.md) · [`UIPropertiesBox.h`](../PropertiesBox/UIPropertiesBox.h.md) · [`UIXmlInitBase.cpp`](../XML/UIXmlInitBase.cpp.md)
**Tier floor** — T3.

## Purpose

Declares the surface implemented in [`UIListBox.cpp`](UIListBox.cpp.md).

The toolkit has two list widgets and they are not variants of each other. This one — the list
*box* — is a [`CUIScrollView`](../ScrollView/UIScrollView.h.md) with all of that class's
pixel-accurate scrolling, variable row heights and selection; the other,
[`CUIListWnd`](../ListWnd/UIListWnd.h.md), is an older row-paged design that shows a whole
number of fixed-height rows. New code uses this one; the shipped screens use both.

## Exported units

- `CUIListBox` — the list.
- `AddItem` / `AddTextItem(text)` / `AddExistingItem(item)` — create a row with the list's
  default height, width, font, text colour and selection texture, or adopt one already built.
  The text form runs the string through the localization table.
- Lookup: `GetItemByTAG`, `GetIdxByTAG`, `GetItemByIDX`, `GetItemByText`, `GetSelectedItem`,
  `GetSelectedText`, `GetText(idx)`, `GetSelectedIDX`.
- Selection: `SetSelectedIDX`, `SetSelectedTAG`, `SetSelectedText`, `SetImmediateSelection`.
- Ordering: `MoveSelectedUp`, `MoveSelectedDown`.
- Appearance defaults applied to rows created afterwards: `SetSelectionTexture`,
  `SetItemHeight`, `SetTextColor`, `SetFont`.
- `GetLongestLength` — the widest row's measured text width, used by callers that size a
  popup to its content.

**Notes** — the appearance setters do not touch existing rows. Order matters: set the
defaults, then add.
