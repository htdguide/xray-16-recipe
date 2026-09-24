# src/xrUICore/ListWnd/UIListItem.cpp

> A button that starts un-pressed, ungrouped and auto-deleted, highlights its text while the pointer is over it, and indents its label past its own icon.

**Needs** — [`UIListItem.h`](UIListItem.h.md) · [`Buttons/UIButton.h`](../Buttons/UIButton.h.md) · [`Lines/UILines.h`](../Lines/UILines.h.md) · [`ui_focus.h`](../ui_focus.h.md)
**Used by** — [`UIListItem.h`](UIListItem.h.md)
**Tier floor** — T3.

## Purpose

Almost all of the row's behaviour is the button's. What is decided here is the default
highlight rule and the label indent.

## State

```text
RECORD ListItem EXTENDS Button
  data          : opaque pointer
  value         : int
  index         : int   # -1 until placed in a list
  group_id      : int   # -1 means ungrouped; set to index by set_index
  highlight_text: bool  # overridden wholesale by the owning list for a group
```

**Invariants** — a row is auto-delete from construction and registers itself with the focus
system, so directional navigation can land on it.

## `init_texture`

**Contract** — loads the row's texture through the button, then shifts the label right by the
texture rectangle's width. The row's graphic is therefore always drawn at the left edge with
the text beginning after it, and a row with no texture has no indent.

## `is_highlight_text`

**Contract** — the default rule: the text is highlighted while the pointer is over the row.
The stored flag set by `SetHighlightText` is *not* consulted by this default — the list
window sets that flag for a whole group and then a subclass that overrides this method reads
it. The base class's own answer ignores it.

**Notes** — this split is confusing and is worth restating: the base row highlights on hover
only; the group-highlight flag exists for subclasses. A rebuild should make one rule, reading
`hover OR group_highlighted`, which is what every shipped subclass effectively does.

## `init_list_item`

**Contract** — sets the row's rect. Called by the list window with the row's computed slot.
