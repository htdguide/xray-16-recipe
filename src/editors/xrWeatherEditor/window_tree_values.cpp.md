# src/editors/xrWeatherEditor/window_tree_values.cpp

> Turns a flat list of backslash-separated names into a browsable tree, and refuses to return a folder as an answer.

**Needs** — [`window_tree_values.h`](window_tree_values.h.md) · [`property_string_values_value_base.hpp`](property_string_values_value_base.hpp.md)
**Used by** — [`window_tree_values.h`](window_tree_values.h.md)
**Tier floor** — T3: string splitting and a tree widget.

## Purpose

A flat list of a few hundred names like `effects\flare`, `fx\fx_sun` and
`particles\weather\rain_heavy` is unusable as a list and obvious as a tree. This file does
the conversion, and it is the only place in the tool that knows names have structure.

## State

See [`window_tree_values.h`](window_tree_values.h.md).

## `values` — building the tree

```text
FUNCTION values(names : list<text>, current : text)
  echo = current
  result = current
  begin a batch update
  clear the tree; clear the selection
  selected = none

  FOR EACH name IN names
    node = none
    remainder = name
    WHILE remainder IS NOT empty
      segment, remainder = split remainder AT the first backslash
      IF node IS none
        node = the root child whose label is segment, IF one exists
        IF node WAS found THEN CONTINUE
        node = add a root child labelled segment, icon = closed folder
      ELSE
        child = the child of node whose label is segment, IF one exists
        IF child EXISTS
          node = child
        ELSE
          node.icon = closed folder        # it has children now
          node = add a child of node labelled segment
    IF node.full_path == current
      selected = node
    node.icon = leaf                        # the last segment is a name, not a folder

  sort the tree: folders before leaves, each group alphabetically
  select selected
  end the batch update
```

**Contract** — splits every name on backslashes and merges the segments into a tree,
reusing nodes that already exist. Marks the node whose full path equals the current choice
and selects it. Never fails; an empty list yields an empty tree.

**Invariants** — a node's icon is the leaf icon if and only if some name ends there.
**That is the load-bearing invariant**: it is how the dialog distinguishes a name from a
folder, and it is set by the *last* segment of every name, so a node that is both a name
and a prefix of other names ends as a leaf.

**Notes** — Three things deserve to survive.

**The separator is a backslash, and it is the format's, not the platform's.** These names
come from the game's own data files and use backslashes regardless of the host system. A
rebuild must split on the separator the *data* uses.

**A node's identity is its full path**, which is how the pre-selection matches and how the
answer is read back. The tree is therefore not just a display — it is a reversible encoding
of the list.

**Folders sort before names.** The sort rule is explicit: a node with children outranks a
node without; two of the same kind compare by label. That puts the structure at the top of
each level and the choosable names below it, which is what makes a three-hundred-entry tree
navigable.

The comparison returns "equal" when a folder is compared with a folder that has children on
both sides, which is not a consistent ordering — two folders never swap, so their relative
order is whatever the insertion left. Harmless in practice, since insertion order follows
the source list, but a rebuild should compare folder labels properly.

## Selecting

```text
ON node_clicked(left)
  node = the node under the pointer, ELSE RETURN
  IF node has children
    echo = ""; result = ""            # a folder is not an answer
  ELSE
    echo = node.full_path; result = echo

ON node_double_clicked(left)
  ... the same, and then accept the dialog

ON node_expanded  : node.icon = open folder
ON node_collapsed : node.icon = closed folder
```

**Contract** — clicking a name chooses it; clicking a folder clears the choice. Double
clicking a name chooses it and closes the dialog.

**Notes** — **Clicking a folder clears rather than being ignored**, which is deliberate:
it lets an author cancel a choice without cancelling the dialog, and it means the echo box
always shows exactly what will be returned. A rebuild should keep the property — the echo
is the answer — rather than the gesture.

The expand and collapse handlers only swap the folder icon; they are the one piece of this
file that is pure decoration.
