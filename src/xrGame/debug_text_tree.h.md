# src/xrGame/debug_text_tree.h

> Declares the on-screen debug text tree implemented in [`debug_text_tree.cpp`](debug_text_tree.cpp.md) and [`debug_text_tree_inline.h`](debug_text_tree_inline.h.md), plus the value-to-text conversions its lines are built from.

**Needs** — [`debug_text_tree_inline.h`](debug_text_tree_inline.h.md)
**Used by** — [`debug_text_tree.cpp`](debug_text_tree.cpp.md) · [`debug_text_tree_inline.h`](debug_text_tree_inline.h.md) · [`level_debug.cpp`](level_debug.cpp.md) · [`level_debug.h`](level_debug.h.md)
**Tier floor** — T3: a declaration plus scalar formatting

## Purpose

Declares `text_tree`, the structure the AI debugging overlays build their output into, and
the two sinks that consume it. Substance is in
[`debug_text_tree.cpp`](debug_text_tree.cpp.md) (column measurement and the two sinks) and
[`debug_text_tree_inline.h`](debug_text_tree_inline.h.md) (building and emitting).

Exported units:

- `text_tree` — a node: a list of column strings, a list of children, a separator character,
  a visibility flag and a group identifier. Copying is forbidden; children are owned.
- `find_node` / `find_or_add` — depth-first search by first column, and the same with
  creation on miss. This is what lets several unrelated debug modules contribute to one
  named section of the overlay without coordinating.
- `add_text` / `add_line` — append columns to this node, and create a child node with one to
  five columns.
- `output` — measure, then walk, handing each formatted line to a caller-supplied sink.
- `toggle_show` / `clear` — visibility by group, and reset.
- `draw_text_tree` — the on-screen sink: alternating row colours, a skip count for
  scrolling.
- `log_text_tree` — the log-file sink, at a fixed indent of two.

## `make_xrstr`

**Contract** — an overload set converting anything a debug line wants to print into text: a
format string with arguments, a boolean, the float and integer widths, a vector, and a
string (identity). Total; no failure path.

**Invariants** — a boolean renders as `+` or `-`, not as a word. The overlay is dense and
one character per flag is the point.

**Notes** — the variadic form formats into a four-kilobyte scratch buffer with no truncation
check, which is a debug-build hazard a rebuild removes by using a growable formatter. The
existence of the overload set at all is a C++ artifact: it is `to_text`, and a rebuild in a
language with a display protocol deletes it.
