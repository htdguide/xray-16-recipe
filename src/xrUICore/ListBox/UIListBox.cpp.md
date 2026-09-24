# src/xrUICore/ListBox/UIListBox.cpp

> Manufactures uniform rows from the list's defaults, forwards their selection and click notifications up as the list's own, and translates the mouse wheel into a scroll step.

**Needs** — [`UIListBox.h`](UIListBox.h.md) · [`UIListBoxItem.h`](UIListBoxItem.h.md) · [`ScrollView/UIScrollView.h`](../ScrollView/UIScrollView.h.md) · [`ScrollBar/UIScrollBar.h`](../ScrollBar/UIScrollBar.h.md) · [`Static/UIStatic.h`](../Static/UIStatic.h.md) · [`UIMessages.h`](../UIMessages.h.md) · [`xrEngine/StringTable/StringTable.h`](../../xrEngine/StringTable/StringTable.h.md)
**Used by** — [`UIListBox.h`](UIListBox.h.md)
**Tier floor** — T3.

## Purpose

The scroll view underneath already does layout, scrolling and selection. What this file adds
is (a) a row factory so every row in one list looks the same, (b) a message relay so a screen
listens to the *list* rather than to each row, and (c) row lookup by index, by tag and by
text.

## State

```text
RECORD ListBox EXTENDS ScrollView
  font               : Font
  default_row_height : real     # 20 virtual units unless set
  text_colour        : colour   # opaque black unless set
  selection_texture  : optional<text>
  immediate_selection: bool     # hovering a row selects it
```

**Invariants**

- Rows are selectable — the scroll view's selectable flag is forced on at construction.
- The scroll bar is the non-fixed kind: the list sizes its thumb from the content.
- The defaults above are read only when a row is created. Changing them later leaves existing
  rows alone.

## `add_item` / `add_text_item` / `add_existing_item`

**Contract** — build a row at the list's default height, five units narrower than the list's
usable width, with the list's font, text colour and either the configured selection texture or
the built-in default one; point its message target at the list; and hand it to the scroll view
with auto-delete. The text form additionally translates the string through the localization
table and sets it. The adopt form does the same dressing to a row the caller built, so a
screen can subclass the row type.

**Notes** — three different widths are set on a new row in sequence — the row's own size, its
text field's width, and then the row's width again. Two of them are the usable width and one
is five units less. The net effect is a row exactly as wide as the usable width whose text
field is also that wide; the intermediate value is dead. The five-unit inset is the visual gap
to the scroll bar and exists in the shipped look.

## `send_message`

**Contract** — when a message arrives from a row (that is, from a child of the scroll view's
content pad), re-send it upward as coming from the *list*: a row selection becomes a list
selection, a row click becomes a list click. Additionally, when a row reports that the pointer
entered it and the list is in immediate-selection mode, select that row. Every message is then
also passed to the scroll view's own handler.

**Notes** — the relayed messages carry the row's payload unchanged, which for selection and
click is a pointer to the row's tag. A screen therefore learns *which* row without holding
row pointers. Immediate selection is what makes a drop-down list highlight-follows-pointer
rather than click-to-select.

## `on_mouse_action`

**Contract** — the wheel scrolls by one step in either direction and is consumed; every other
action goes to the plain window dispatch. Note that the *window* dispatch is used, not the
scroll view's — the scroll view's own mouse handling is bypassed.

## Lookup and selection

**Contract** — `GetItemByIDX` counts rows from the top; `GetItemByTAG` matches a row's
caller-assigned tag; `GetItemByText` matches the row's text exactly, case-sensitively;
`GetSelectedIDX` reports the position of the selected row or the all-ones sentinel when
nothing is selected. Each is a linear walk of the content pad, filtered to children that
really are list rows — a pad can contain other widgets.

**Notes** — `GetIdxByTAG` differs from the others: it returns the index of the row with the tag
*or*, if no row has it, the index of the last row. It starts the counter at -1, promotes it to
0 on the first row seen, and increments thereafter, so the value returned when nothing matched
is the last index rather than a failure. Callers treat it as a search; it is not one. Combo
box selection by token id goes through it, which is why selecting an unknown token silently
selects the last entry.

## `move_selected_up` / `move_selected_down`

**Contract** — swap the selected row with its neighbour in the content pad's child list and
force a re-layout. Does nothing when nothing is selected or the row is already at the end.
Reordering is by swapping list positions; row indices are positional, so every index changes
implicitly.

## `get_longest_length`

**Contract** — measures every row's text with that row's own font and returns the widest, in
virtual units. Used by popups that size themselves to their content.
