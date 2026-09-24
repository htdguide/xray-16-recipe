# src/xrUICore/ComboBox/UIComboBox.cpp

> A drop-down that probes two generations of texture names to dress itself, grows to hold its list while open, locks focus and captures the mouse for the duration, draws the open list after the rest of the UI, and maps the selection onto a console token.

**Needs** — [`UIComboBox.h`](UIComboBox.h.md) · [`ListBox/UIListBox.h`](../ListBox/UIListBox.h.md) · [`ListBox/UIListBoxItem.h`](../ListBox/UIListBoxItem.h.md) · [`ScrollBar/UIScrollBar.h`](../ScrollBar/UIScrollBar.h.md) · [`XML/UITextureMaster.h`](../XML/UITextureMaster.h.md) · [`Options/UIOptionsItem.h`](../Options/UIOptionsItem.h.md) · [`ui_focus.h`](../ui_focus.h.md) · [`UIMessages.h`](../UIMessages.h.md) · [Data: Configuration](../../../SYSTEM-REQUIREMENTS.md#5-data-and-persistence)
**Used by** — [`UIComboBox.h`](UIComboBox.h.md)
**Tier floor** — T3.

## Purpose

A drop-down is the widget where the toolkit's flat z-order hurts most, and most of this file
is the workaround. It is also the one widget whose appearance had to be made to work against
two games' texture sets without a data migration.

## State

```text
RECORD ComboBox EXTENDS Window, OptionsItem
  frame_line   : InteractiveBackground   # the closed line's four-state background
  text         : Static                  # the closed line's label
  list_frame   : FrameWindow             # the expanded list's border; its visibility IS the open flag
  list         : ListBox                 # a child of list_frame
  state        : ENUM { expanded, folded }
  list_rows    : int                     # 4 unless set
  token_id     : int                     # the selected entry's identifier
  backup_token : int
  disabled_ids : set<int>
  colour_normal, colour_disabled : colour
  initialized  : bool
```

**Invariants**

- The frame's visibility *is* the open state; the separate state field mirrors it and exists
  only for the mouse handler's switch.
- Entries carry their identifier in the row's opaque data field, not in its tag. Selection by
  identifier therefore goes through the list's index-by-tag lookup — which, as noted in
  [`UIListBox.cpp`](../ListBox/UIListBox.cpp.md), returns the last index rather than failing
  when nothing matches.
- Adding an entry before initialization is refused outright.

## `init_combo_box`

**Contract** — builds the whole composition and sizes everything from one width. Row height is
not configured: it is read from a *texture's* height, probed under two names.

```text
FUNCTION init_combo_box(pos, width)
  TEXT_INSET <- 5

  self.rect <- (pos, width × 20)
  frame_line.init(origin, width × 20)

  # two generations of the game name this texture differently.
  # probe the newer name non-fatally; on success the highlighted state
  # exists too, otherwise fall back to the older pair.
  IF frame_line.init_state(enabled, "ui_inGame2_combobox_linetext", non_fatal)
    frame_line.init_state(highlighted, "ui_inGame2_combobox_linetext")
  ELSE
    frame_line.init_state(enabled,     "ui_cb_linetext_e", non_fatal)
    frame_line.init_state(highlighted, "ui_cb_linetext_h", non_fatal)

  text.rect <- (TEXT_INSET, 0, width - TEXT_INSET, 20)
  text.vertical_align <- centre ; text.colour <- colour_normal ; text.enabled <- false

  row_height <- height_of_texture("ui_inGame2_combobox_line_b")
             OR height_of_texture("ui_cb_listline_b")

  list.rect <- (TEXT_INSET, 0, width - TEXT_INSET, row_height * list_rows)
  list.row_height <- row_height
  list.selection_texture <- whichever of the two generations' row textures exists
  list_frame.texture     <- whichever of the two generations' frame textures exists
  list_frame.rect <- (0, 20, width, row_height * list_rows)
  list.visible <- true ; list_frame.visible <- false
  list.message_target <- self
```

**Notes** — the closed height and the text inset are both fixed constants; only the row height
comes from data, and it comes from the *height of a texture entry*, not from an attribute.
That is the single most surprising dependency in the widget: replacing the drop-down art with
art of a different height silently changes the widget's open height.

The text static is disabled, so it never takes a hit test — the whole closed line belongs to
the combo box.

## `show_list`

**Contract** — opening grows the widget's height to the closed line plus the list, shows the
frame, asks the parent to route all mouse input here regardless of position, and locks the
focus system to the frame so directional navigation cannot leave the open list. Closing
reverses all four.

**Notes** — the mouse capture is what lets a click *anywhere* close the list, including outside
the widget. The focus lock is the keyboard and gamepad equivalent. Both must be released on
close or the UI is stuck; the close path is reached from six places, which is why every one of
them goes through this one function.

## `update`

**Contract** — per frame: a disabled combo box shows the disabled background state and the
disabled text colour; an enabled one shows the normal text colour and, if the list is open,
re-registers itself in the device's render sequence at a late priority.

**Notes** — the re-registration is a remove-then-add every frame while open. The priority
places the draw after the ordinary UI pass; `OnRender` then draws the frame and immediately
removes the registration, so the deferral is one frame at a time. That is the whole
"draw above everything" mechanism for the drop-down, and it is a workaround for the absence of
a z-ordered overlay layer.

## `on_render`

**Contract** — if the widget is shown and the list is open, draw the list frame and then
unregister from the render sequence. Draws nothing otherwise.

## `on_mouse_action`

**Contract** — offers the action to children first. Then, on a left press: if the list is open
and the press was *not* over the scroll bar, close it and consume; otherwise toggle the open
state and consume. On a right press, advance the selection by one with wrap-around.

**Notes** — the open case deliberately falls through into the folded case when the press *was*
over the scroll bar, so dragging the thumb does not close the list. Right-clicking to cycle
without opening is a shipped convenience.

## `on_keyboard_action` / `on_controller_action`

**Contract** — while the pointer is over the widget: the accept and back actions close an open
list; left and right cycle the selection without wrapping *only while closed*; up and down
cycle it *only while open*, and additionally move the focus system onto the newly selected row
so the highlight follows. The gamepad path is the same rule keyed off a two-axis stick with a
0.5 activation threshold and a 0.2 cross-axis dead zone.

**Notes** — the thresholds are the standard "don't fire diagonals" pair used throughout the
toolkit's controller handling. The left/right-when-closed and up/down-when-open split is what
makes a settings page navigable without ever opening a list.

## `on_list_item_select`

**Contract** — when the list reports a click, copy the row's text into the closed line, read the
row's identifier, close the list, and — only if the identifier actually changed — announce a
selection upward.

## `set_current_opt_value` and the other settings operations

**Contract** — `set-current` rebuilds the entry list from the bound console variable's token
table, skipping identifiers on the disabled set, then selects the entry whose *translated*
text matches the variable's current token name, falling back to identifier 1 if none matches.
`save` writes the selected identifier's token name back through the console. `save-backup`,
`undo` and `is-changed` work on the identifier, not the text.

**Notes** — matching by translated text rather than by identifier is fragile: two tokens whose
localized names collide are indistinguishable, and a missing translation matches nothing. The
fallback identifier of 1 is a guess at "the first real entry" and is not derived from the token
table. Recorded as behaviour to preserve but not to admire.

## `set_next_item_selected`

**Contract** — moves the selection one entry forward or backward, wrapping or stopping at the
end as asked, and reports whether it moved. Selecting an entry always re-reads its identifier
and re-renders the closed line, and runs the settings item's apply-on-change hook — so a combo
box marked apply-on-change writes the console variable on every cycle, not only on commit.
