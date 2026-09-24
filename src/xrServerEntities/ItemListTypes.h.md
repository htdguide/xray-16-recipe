# src/xrServerEntities/ItemListTypes.h

> One row in an editor list: a key, a type tag, a payload and the callbacks that let the host respond to clicks and draw a thumbnail.

**Needs** — [`PropertiesListTypes.h`](PropertiesListTypes.h.md)
**Used by** — [`xrEProps.h`](xrEProps.h.md)
**Tier floor** — T3: an editor widget's data model.

## Purpose

The level editor's object browsers are all "a list of named things"; this is that row. It is
in this directory only because the entity records are what get listed, and the editor and
the game must agree on the shape. Nothing in the game reads it.

## State

```text
RECORD ListRow
  key         : text           # the displayed and searched name
  type        : int            # caller-defined; distinguishes folders from leaves
  payload     : opaque         # the thing this row stands for
  object      : opaque         # a second, caller-owned association
  tag         : int
  icon_index  : int            # -1 when there is none
  colour      : int (32-bit)
  flags       : int (32-bit bitfield)
      show_checkbox = bit 0    checked = bit 1
      draw_thumbnail = bit 2   draw_canvas = bit 3
      sorted = bit 4           hidden = bit 5
  on_click, on_focus, on_draw_thumbnail : callbacks
```

**Invariants** — visibility is stored as a *hidden* bit, so a freshly zeroed row is visible.
That is the only non-obvious thing in the record.

## Notes

The two opaque payload fields are a duplication: one is set through the constructor and
reachable through an accessor, the other is a public field the host fills in directly. There
is no rule about which to use. A rebuild has one.
