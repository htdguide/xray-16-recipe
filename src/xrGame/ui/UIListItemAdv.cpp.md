# src/xrGame/ui/UIListItemAdv.cpp

> A list row that grows rightwards: each field is appended at the running sum of its predecessors' widths, so a row is authored by a sequence of widths rather than by positions. Excluded from the build.

**Needs** — [`UIListItemAdv.h`](UIListItemAdv.h.md) · [`UIListItem.h`](../../xrUICore/ListWnd/UIListItem.h.md)
**Used by** — [`UIListItemAdv.h`](UIListItemAdv.h.md)
**Tier floor** — T3.

## Purpose

A multi-column list row. It is **commented out of the build** and superseded by the server
browser's row, which does the same thing with explicit offsets. It is worth a page for the one
idea it states cleanly: **a row's columns are defined by widths, not by positions.**

## `AddField` / `AddWindow`

**Contract** — append a text field of a given width, or an arbitrary widget, at the row's
current right edge. A text field takes the row's full height, the row's font and the row's
current text colour, and is owned by the row. An arbitrary widget keeps its own size and is
**centred vertically** in the row.

```text
FUNCTION next_left_edge() -> real
  RETURN the sum of every existing child's width
```

**Invariants** — the running edge is computed from **every child**, not only from the fields
this class tracks. A row that inherited children from its base therefore starts after them,
which is exactly what is wanted: the base row's own text item occupies the first column.

**Notes** — recomputing the sum on every append is quadratic in the column count and entirely
irrelevant at six columns. What matters is that the sum is *derived*, not stored: a column
whose width changes after the fact does not shift the ones after it, which is a limitation and
is why the surviving implementation passes explicit offsets instead.

## `SetTextColor`

**Contract** — recolours the row's own text and every appended field together. Appended
widgets that are not text fields are untouched.

**Notes** — this is the whole reason the class tracks its fields separately from its children.
A list recolours a row to show selection, and the row must push that down to columns the list
never saw.
