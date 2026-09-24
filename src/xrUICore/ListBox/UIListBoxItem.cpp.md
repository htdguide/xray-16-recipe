# src/xrUICore/ListBox/UIListBoxItem.cpp

> A row that draws its highlight only while selected, selects itself on press or on gaining pointer focus, and appends fields at its running right edge.

**Needs** — [`UIListBoxItem.h`](UIListBoxItem.h.md) · [`ScrollView/UIScrollView.h`](../ScrollView/UIScrollView.h.md) · [`Static/UIStatic.h`](../Static/UIStatic.h.md) · [`ui_focus.h`](../ui_focus.h.md) · [`UIMessages.h`](../UIMessages.h.md) · [Seam: Windowing and input](../../../SYSTEM-REQUIREMENTS.md#seam-windowing-and-input)
**Used by** — [`UIListBoxItem.h`](UIListBoxItem.h.md)
**Tier floor** — T3.

## Purpose

The row is where "selected" becomes visible and where a pointer or key lands. The interesting
decisions are the two selection paths and the fact that the row reaches its owning scroll view
by walking two levels up the tree.

## `draw`

**Contract** — draw the stretched-line background *only* when the row is selected, then draw
the fields. An unselected row is therefore transparent and the list's own background shows
through.

## `on_mouse_down`

**Contract** — a left press selects the row in the owning scroll view and announces both a
selection and a click, carrying the row's tag; a right press announces a right-press with the
tag. Both consume. Other buttons are ignored.

**Notes** — the owning scroll view is found as the parent's parent, because the scroll view
interposes a content pad between itself and its rows. That two-step walk is a structural
assumption a rebuild should replace with an explicit owner reference; it silently breaks for
any row placed anywhere else.

## `on_focus_receive`

**Contract** — when the pointer enters the row, announce that focus was received. Then, *only
if the focus system agrees this row is the focused one*, also select it in the scroll view and
announce a selection.

```text
FUNCTION on_focus_receive()
  inherited.on_focus_receive()
  message_target.send(self, LIST_ITEM_FOCUS_RECEIVED)
  IF focus_system.focused == self
    owning_scroll_view.set_selected(self)
    message_target.send(self, LIST_ITEM_SELECT, tag)
```

**Notes** — the guard is what separates mouse hovering from directional navigation. Hovering
sets the pointer-over flag and fires this, but the focus system's current element is only
changed by keyboard or gamepad navigation, so hovering announces focus-received (which the
list box turns into a selection when it is in immediate-selection mode) while navigating
actually selects. Two mechanisms with similar names doing different things; the distinction
is load-bearing.

## `on_keyboard_action`

**Contract** — when the row is both under the pointer and selected, the key bound to the UI
*accept* action announces a click with the row's tag and consumes. Otherwise the key goes to
children.

## `add_text_field` / `add_icon_field`

**Contract** — appends a field at the row's current right edge, full row height, at the given
width. A text field additionally inherits the row's font and text colour, is vertically
centred, and is given its text.

```text
FUNCTION fields_right_edge() -> real
  IF no children RETURN 0
  last <- the most recently attached child
  RETURN last.x + last.width
```

**Notes** — the right edge is the *last* child's right edge, not the maximum over children, so
fields must be appended in left-to-right order and a field added out of order lands on top of
its predecessor. The original once summed over all children and that form is commented out;
the surviving behaviour is the one to reproduce.

## Construction

**Contract** — sets the row height, appends one 10-unit-wide text field, and registers with the
focus system. The initial field width is immediately overwritten by the list box, which sizes
it to the list; the constant only matters for a row built by hand.
