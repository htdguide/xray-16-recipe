# src/xrGame/debug_text_tree.cpp

> Measures a debug text tree into aligned columns, and provides the two sinks that render it — to the screen in alternating colours, or to the log.

**Needs** — [`debug_text_tree.h`](debug_text_tree.h.md) · [`Level.h`](Level.h.md) · [`xrUICore/ui_base.h`](../xrUICore/ui_base.h.md) · [`xrEngine/GameFont.h`](../xrEngine/GameFont.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: string measurement and text output

## Purpose

The AI debugging overlays produce deeply nested, tabular information: a creature, its
brain, its current goal, its plan, each action's preconditions and their truth. Printing
that as free text is unreadable. This structure gives it two properties that make it
readable — **nesting by indentation** and **columns that line up across the whole tree,
regardless of depth**.

The second property is the one that costs something, and it is why output is a two-pass
operation: the widths cannot be known until the entire tree has been walked.

## State

```text
RECORD TextTree
  strings      : list<text>        # the columns of THIS line
  children     : list<TextTree>    # owned; order is emission order
  separator    : text (1 char)     # printed between columns, default ':'
  shown        : bool              # false hides this node and its whole subtree
  group_id     : int               # for bulk visibility toggling
  num_siblings : int               # filled by the measuring pass: the size of the
                                   #   subtree rooted here, including itself
```

**Invariant** — a node is not copyable and owns its children outright. A tree is built,
emitted and cleared; it is never shared.

**Invariant** — `num_siblings` is output of the measuring pass, not input. It counts only
*shown* descendants, so a sink can use it to decide how much room a subtree needs before
drawing it. Nothing in this file uses it; both sinks receive and ignore it.

## `prepare` — the measuring pass

**Contract** — recursive; computes the subtree size of every shown node and grows a shared
column-width table so that column *n* is wide enough for the widest column *n* anywhere in
the tree. Mutates the tree (the subtree counts) and the table. Must run before emission.

```text
FUNCTION prepare(node, current_indent, indent_step, columns)
  node.num_siblings = 1
  FOR EACH child IN node.children WHERE child.shown
    prepare(child, current_indent + indent_step, indent_step, columns)
    node.num_siblings = node.num_siblings + child.num_siblings

  grow columns to at least the number of strings on this line

  # A line with a single string is a HEADING, not a row: it must not stretch a column.
  IF node.strings has more than one entry THEN
    FOR EACH (i, s) IN node.strings
      width = length(s) + (i == 0 ? current_indent : 0)
      columns[i] = max(columns[i], width)
```

**Invariants** — the indent is folded into the *first* column's width rather than added
outside the table. That is what makes columns two and beyond line up across different depths:
a deeply indented line's first field is shorter by exactly the extra indentation, so all
second fields still start at the same screen position.

**Invariants** — single-string lines are excluded from the measurement, so a long section
heading does not push every row's second column off the screen. The emission pass makes the
matching exclusion when padding.

**Notes** — the table is one shared vector for the whole tree, deliberately. Per-depth
column tables would be the obvious design and would give a ragged overlay.

## `find_node` · `find_or_add`

**Contract** — `find_node` is a depth-first search for the first node whose *first* column
equals the given text, checking itself before its children; it returns nothing on a miss.
`find_or_add` returns the found node or appends a new child line with that text.

**Invariants** — matching is on the first column only and is exact. The first column is
therefore the node's name, and a debug module that wants to contribute to a section must
spell the section's name identically. This is the whole coordination mechanism between
independent overlays; there is no registry.

**Notes** — the search is linear over the whole tree per call, and the overlays call it once
per section per frame. The trees are small enough that this has never mattered.

## `toggle_show`

**Contract** — flips this node's visibility if its group identifier matches. **Not
recursive**, and nothing in the file ever assigns a non-zero group: every node created by
the line-adding path takes the default group. So group-based toggling is, as shipped,
inoperative except on a root the caller constructed with an explicit group. A rebuild that
wants the feature must recurse and must let `add_line` inherit the parent's group.

## `add_line` (root form) · `clear`

**Contract** — appends an empty child and returns it, and the destructive reset that deletes
every child recursively. `clear` also runs from the destructor.

## `draw_text_tree`

**Contract** — the on-screen sink. Positions the statistics font at the given origin, then
emits one line per shown node, alternating between two colours by row parity so that dense
output stays readable. `offs` skips that many lines from the top, which is how the overlay
scrolls.

**Invariants** — the parameters are pushed into a **process-wide mutable block** before the
walk, because the sink is a callable that must be copyable and stateless. Two overlays
cannot draw concurrently, and the helper is not reentrant. This is an artifact of the sink
being a function object passed by value; a rebuild passes a closure and the global vanishes.

**Notes** — `column_size` and `max_rows` are accepted and stored and then never read: the
multi-column wrapping they served is commented out. A rebuild should drop both parameters
or finish the feature.

## `log_text_tree`

**Contract** — the log sink, at a fixed indentation of two spaces. Emits each line to the
engine log.

**Notes** — the line is passed to the log as a *format string* rather than as an argument,
so any percent sign in debug text — a percentage in a condition readout, say — is
interpreted as a conversion. That is a live defect in a debug path.
