# src/xrGame/debug_text_tree_inline.h

> Building a debug text tree line by line, and the emission pass that pads each line into the measured columns.

**Needs** — [`debug_text_tree.h`](debug_text_tree.h.md)
**Used by** — [`debug_text_tree.h`](debug_text_tree.h.md)
**Tier floor** — T3: string assembly

## Purpose

The half of [`debug_text_tree.cpp`](debug_text_tree.cpp.md)'s structure that has to be
visible to every caller: adding columns and lines, and the emission walk. In the original
these are templates — over the column value type and over the sink type — which is a C++
reason for the split and not a design one.

## State

Adds nothing.

## `add_text` · `add_line`

**Contract** — `add_text` converts one value to text (see `make_xrstr` in
[`debug_text_tree.h`](debug_text_tree.h.md)) and appends it as a column of *this* node.
`add_line` creates a **child** node and fills it with one to five columns, returning the
child so the caller can keep adding to it or hang further children beneath it.

The distinction is the whole vocabulary of the structure: `add_text` widens a row,
`add_line` deepens the tree. The one-to-five forms are a fixed-arity family only because the
original predates variadic argument packs; a rebuild has one variadic function.

## `output` — the emission pass

**Contract** — runs the measuring pass over the whole tree, then walks it emitting one
formatted line per shown node to the sink. Takes the indentation step in spaces. Allocates a
buffer per line.

```text
FUNCTION emit(node, current_indent, indent_step, columns, sink)
  buffer = current_indent spaces
  FOR EACH (i, s) IN node.strings
    buffer += s
    padding = columns[i] - length(s)
    IF i == 0 THEN padding = padding - current_indent    # indent was folded into column 0
    IF node has exactly one string THEN padding = 0      # a heading is not padded
    buffer += padding spaces
    IF s is not the last THEN buffer += " " + separator + " "
  IF node has any strings THEN sink(buffer, node.num_siblings)

  FOR EACH child IN node.children WHERE child.shown
    emit(child, current_indent + indent_step, indent_step, columns, sink)
```

**Invariants** — the two exclusions here must mirror the two in the measuring pass exactly.
Subtracting the indent from column zero's padding is what keeps later columns aligned across
depths; skipping padding for a single-column line is what keeps headings from being padded
to a width they never contributed to.

**Invariants** — children are emitted after their parent's own line, depth first, in
insertion order. The overlay's reading order is the insertion order, so the order debug code
adds lines in is the order they appear.

**Notes**

- A node with no columns emits nothing but its children still emit. That is how a pure
  grouping node works.
- The padding count is computed in an unsigned width, so a column that is *narrower* than
  its measured width — which cannot happen if the measuring pass ran over the same tree —
  would wrap to an enormous count and hang the line. The invariant that measurement precedes
  emission over an unmodified tree is what keeps that unreachable, and a rebuild should
  clamp instead of relying on it.
- The buffer gets an explicit terminator appended before being handed to the sink, an
  artifact of the sink taking a C string. Immaterial to a rebuild.
