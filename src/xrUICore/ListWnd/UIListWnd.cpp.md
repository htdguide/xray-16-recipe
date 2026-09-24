# src/xrUICore/ListWnd/UIListWnd.cpp

> Shows a whole number of fixed-height rows from a row index, keeps rows outside that window hidden rather than clipped, propagates click and focus across a row's whole group, and follows directional navigation by scrolling the focused row into view.

**Needs** — [`UIListWnd.h`](UIListWnd.h.md) · [`UIListWnd_inline.h`](UIListWnd_inline.h.md) · [`UIListItem.h`](UIListItem.h.md) · [`ScrollBar/UIScrollBar.h`](../ScrollBar/UIScrollBar.h.md) · [`Windows/UIFrameLineWnd.h`](../Windows/UIFrameLineWnd.h.md) · [`Cursor/UICursor.h`](../Cursor/UICursor.h.md) · [`ui_focus.h`](../ui_focus.h.md) · [`UIMessages.h`](../UIMessages.h.md) · [`ui_base.h`](../ui_base.h.md)
**Used by** — [`UIListWnd.h`](UIListWnd.h.md)
**Tier floor** — T3.

## Purpose

Three ideas carry this file.

*Paging instead of clipping.* The visible window is a contiguous run of rows starting at a
scroll index; rows in it are shown and positioned at slot heights, rows outside it are hidden.
Nothing is partially drawn, so there is no scissor rectangle for the rows themselves. That is
why the row height must be uniform and why the widget's height is quietly truncated to a whole
number of rows.

*Groups.* Several rows can share a group identifier and then behave as one row: clicking any of
them selects all of them and announces one click; hovering any of them highlights all of them.
This is how a multi-line entry — an icon row plus a description row — behaves like a single
entry.

*Two cursors.* A focused row (pointer or forced) and a selected row (clicked), each optionally
drawn with its own highlight frame, and only one frame object shared between them.

## State

```text
RECORD ListWnd EXTENDS Window
  rows              : list<ListItem>      # ordered; index == position
  row_height        : real
  row_width         : real
  rows_that_fit     : int                 # floor(height / row_height)
  first_shown_index : int
  focused_index     : int   # -1 for none
  focused_group     : int
  selected_index    : int   # -1 for none
  selected_group    : int
  force_focused     : bool  # a caller pinned the focus; hovering must not move it
  show_selected     : bool
  active_background : bool
  highlight_frame   : optional<FrameLine> # shared by both cursors; exists iff either is on
  scroll            : ScrollBar
  list_active       : bool  # rows accept input
  vert_flip         : bool
  text_colour       : colour
  pending_mouse_move: bool
```

**Invariants**

- `rows_that_fit` is recomputed whenever the height or the row height changes, and *all*
  placement is derived from it.
- A row's index equals its position; every group walk and every highlight test compares
  indices.
- A group identifier of -1 excludes a row from every group walk, so rows that opt out of
  grouping must use it — and the default from `SetIndex` is *not* -1, it is the index itself.
- The highlight frame exists exactly while either cursor's drawing is enabled; enabling either
  creates it, disabling one destroys it only if the other is also off.

## `init_list_wnd`

**Contract** — sets the rect, creates the scroll bar with the configured profile (or the
default one), moves it inside the right edge by its own width, sizes rows to the remaining
width, sets the row height and computes how many rows fit, ranges the scroll bar empty, hides
and disables it, and lays out.

**Notes** — the scroll bar is created at the right edge and then shifted left by its own width,
so it sits *inside* the list rather than beside it, and the rows' width is the list width less
the bar width whether or not the bar is showing. A list with fewer rows than fit therefore has
a permanent gutter. That is the shipped look.

## `update_list`

**Contract** — the layout. Hides every row before the window, places and shows every row in
the window (plus one), and hides every row after it. Placement is by slot index within the
window, not by absolute row index, so scrolling moves the rows rather than the viewport.
Enabled-ness of the visible rows follows the list's activity flag.

```text
FUNCTION update_list()
  IF rows IS empty THEN refresh_scroll_visibility() ; RETURN

  FOR i IN 0 .. min(count, first_shown_index) - 1
    rows[i].visible <- false

  FOR i IN first_shown_index .. min(count, first_shown_index + rows_that_fit + 1) - 1
    slot <- i - first_shown_index
    y    <- IF vert_flip THEN height - slot * row_height - row_height
                         ELSE slot * row_height
    rows[i].rect    <- (rows[i].x, y, width = row_width, height = row_height)
    rows[i].visible <- true
    rows[i].enabled <- list_active

    FOR i IN first_shown_index + rows_that_fit .. count - 1
      rows[i].visible <- false

  refresh_scroll_visibility()
```

**Notes** — the middle loop runs one row past the window and the last loop starts one row
before the window's end, so the boundary row is shown by the middle loop and then hidden by
the last one. The net effect is exactly `rows_that_fit` visible rows, and the overlap is how
the code reaches that; reproduce the *count*, not the double pass.

Rows keep their own horizontal position, which the insertion path set from the row's `shift`.
Only the vertical slot is recomputed.

## `send_message`

**Contract** — the message hub. Three sources.

From the scroll bar: a vertical scroll sets the first shown index from the bar, re-lays out,
and re-announces the scroll upward so an owning screen can react.

From a row, on a click: walk every row, and for each one in the same group as the clicked row,
turn its text highlight on and tell it it is selected; for each one in a different group, turn
the highlight off and tell it it is unselected. Record the clicked row's index and group as
the selection. Then announce one click upward, carrying the clicked row.

From a row, on pointer entry: unless the focus is pinned, record that row's index and group as
the focus; then do the same group walk as for a click, but with focus-received and focus-lost
instead of select and unselect. On pointer exit: clear the focus if it was this row and is not
pinned, drop every row's highlight, tell every row focus was lost, and arm a synthetic mouse
move for the next frame.

**Notes** — rows with group -1 are skipped by every walk, so an ungrouped row never highlights
a neighbour and never gets these notifications. The synthetic mouse move on exit exists because
a row that was hidden or moved out from under the pointer leaves the tree believing the pointer
is still over it; replaying a move at the last known position re-resolves it.

## `draw`

**Contract** — draws the two cursor highlights first, then the rows.

The focus highlight draws one frame per visible row whose group matches the focused group —
so a whole group highlights. The selection highlight draws one frame for the row whose index
matches the selected index, clipped to the list's rectangle.

```text
FUNCTION draw_highlight(list_rect, row)
  y <- list_rect.top + (row.index - first_shown_index) * row_height
  # centre a frame shorter than the row within the row
  slack <- row_height - frame.height
  IF slack > 0 THEN y <- y + floor(slack / 2)
  frame.position <- (list_rect.left, y)
  frame.width    <- list_width - (IF scroll.visible THEN scroll.width ELSE 0)
  frame.draw()
```

**Notes** — only the selection highlight is clipped; the focus highlight is not. A focused row
scrolled partly out of the window would therefore draw its highlight outside the list. In
practice paging makes partial rows impossible, so the asymmetry is unobservable — and it is
the honest reading of why one has a scissor and the other does not.

The frame's width excludes the scroll bar only when the bar is showing, so the highlight
tracks the gutter rather than the row width.

## `update`

**Contract** — per frame: replay a pending synthetic mouse move if one was armed; then, if the
pointer is over the list and the focus system reports a focused widget, find which of this
list's rows that widget lives under, and if it is not the currently selected row, scroll it
into view. If the scroll position actually changed, synthesize a click on that row and warp the
pointer onto the focused widget.

```text
FUNCTION update()
  IF pending_mouse_move
    replay mouse_move at the last cursor position ; pending_mouse_move <- false

  focused <- IF cursor_over_window THEN focus_system.focused ELSE none
  IF focused EXISTS
    row <- the ancestor of focused that is an immediate child of this list
    IF row IS a ListItem AND row != current selected row
      before <- first_shown_index
      scroll_to(index_of(row))
      IF first_shown_index != before
        send_message(row, BUTTON_CLICKED)      # adopt it as the selection
        cursor.warp_to(focused)
```

**Notes** — this is directional navigation's hook into the list. Pointer position remains the
list's only hit-test input, so navigating to a row off-screen must *both* scroll it in and move
the pointer onto it, or the next frame's hit test would undo the focus. Synthesizing the click
only when the scroll actually moved means navigating within the visible window does not select.

## `scroll_to_begin` / `scroll_to_end` / `scroll_to_pos`

**Contract** — set the scroll bar's position, copy it back into the first-shown index, and
re-lay out. `ScrollToEnd` positions so the last row is the last visible one, clamped to the
minimum. `ScrollToPos(index, ratio)` places the given row a fraction of the way down the
window — the default centres it — clamped to the scrollable range, and does nothing when the
scroll bar is disabled.

**Notes** — the clamp expression in `ScrollToPos` contains a stray operator in the original and
the upper bound it computes is not the one the neighbouring functions use. Treat the intent —
clamp to `[min, max - page + 1]` — as the contract.

## `remove_item` / `remove_all` / `get_item` / `find_item` / `find_item_with_value` / `get_item_pos`

**Contract** — removal detaches the row (which releases it, since rows are auto-delete), re-lays
out, re-ranges the scroll bar, and renumbers every remaining row from zero. The lookups are
linear walks returning an index or -1. Removing everything additionally resets the scroll
position and the row cursors.

**Invariants** — renumbering after removal also resets every row's group to its new index, for
the same reason as on insertion. A list that uses groups must re-group after any removal.

## `update_scroll_bar`

**Contract** — decides whether the bar is visible: when the always-show override is enabled, the
bar's visibility is the override's value; otherwise the bar shows exactly when the row count
exceeds the page size.

## `set_height` / `set_item_height` / `set_width` / `set_item_width`

**Contract** — changing the height or the row height recomputes how many rows fit and re-lays
out; changing the height also resizes the scroll bar. Changing the list width does *not* change
the row width — that is a separate setter — which is why a resized list keeps its rows at their
old width until a caller fixes it.

## `enable_active_background` / `show_selected_item` and the frame's lifecycle

**Contract** — either cursor's drawing being turned on creates the shared highlight frame (sized
to the list width by 18 units, with the built-in row-line texture); turning one off destroys the
frame only when the other is also off.

**Notes** — 18 units is the frame's own height and is unrelated to the row height; the draw path
centres it in whatever the row height is. It is not derived from anything discoverable.

## `reset` / `activate_list` / `on_mouse_action` / `on_keyboard_action`

**Contract** — `Reset` returns every row and the list itself to the neutral state. `ActivateList`
gates whether visible rows are enabled, applied at the next layout. The mouse wheel scrolls by
one row and is consumed; everything else, and every key, goes to the ordinary window dispatch.
