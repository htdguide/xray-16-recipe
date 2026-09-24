# src/editors/xrSdkControls/Controls/TreeViewFilterPanel/TreeViewFilterPanel.cs

> Live filtering that *removes* non-matching nodes from the tree and remembers them by path, so a shortened filter can put them back.

**Needs** — [`TreeView.cs`](../TreeView/TreeView.cs.md) · [`TreeNode.cs`](../TreeNode/TreeNode.cs.md) · [`TreeViewFilterPanel.Designer.cs`](TreeViewFilterPanel.Designer.cs.md)
**Used by** — [`TreeView.Designer.cs`](../TreeView/TreeView.Designer.cs.md) · [`TreeView.cs`](../TreeView/TreeView.cs.md) · [`TreeViewFilterPanel.Designer.cs`](TreeViewFilterPanel.Designer.cs.md)
**Tier floor** — T3: string matching and tree surgery.

## Purpose

A text box above every tree. As the author types, the tree narrows to matching entries. Every tree in the editor has one, supplied automatically by the container substitution in [`TreeView`](../TreeView/TreeView.cs.md).

The design decision — and it is a consequential one — is that **filtering detaches nodes rather than hiding them**. The host toolkit's tree has no per-node visibility, so a node that must not be shown must leave the tree. The panel therefore has to remember what it removed, and where from.

## State

```text
RECORD FilterPanel
  tree              : TreeView
  auto_expand       : bool                      # expand the path to each match
  hidden_leaves     : map<text, TreeNode>       # full path -> the detached node
  hidden_folders    : map<text, text>           # full path -> the folder's label
```

**Invariants** — a node is in exactly one of two places: the tree, or the hidden map keyed by the full path it was detached from. The path is the only record of where it belongs, which is why the tree's path walk (which can re-create missing folders) is what restores it.

**Notes** — a detached *folder* is remembered by path and label only, not by object: its children went into the leaf map under their own full paths, so restoring the folder means re-creating it empty and letting its children find their way back. That is why the two maps have different value types, and it is the one genuinely subtle thing in the file.

## Filtering

**Contract** — on every keystroke, first remove from the tree everything that no longer matches, then restore from the hidden maps everything that now does, then re-sort. An empty filter skips the removal pass, so everything is restored.

```text
ON filter_changed(text)
  filter = lowercase(text)
  IF filter is not empty THEN remove_non_matching(tree.nodes, filter)
  restore_matching(filter)
  tree.sort()
```

**Invariants** — removal runs before restoration. The order matters: a node that fails the new filter must leave before a node that passes it arrives, or the intermediate tree contains both and the sort sees a state that was never meant to exist.

## `remove_non_matching`

**Contract** — depth-first over a level's nodes. A leaf whose label does not contain the filter is detached and remembered; one that does is left, and its ancestors are expanded if auto-expand is on. A folder is recursed into first, then detached if it ended up empty *and* its own label does not match — so a folder survives either because it has surviving children or because it is itself a match.

```text
FUNCTION remove_non_matching(nodes, filter)
  FOR EACH node IN nodes           # by position; removal does not skip the successor
    IF node is a leaf
      IF filter NOT IN lowercase(node.label)
        hidden_leaves[node.full_path] = node ; detach node
      ELSE IF auto_expand THEN expand every ancestor of node
    ELSE
      remove_non_matching(node.children, filter)
      IF node.children is empty AND filter NOT IN lowercase(node.label)
        hidden_folders[node.full_path] = node.label
        detach node ; collapse node
```

**Notes** — a folder whose *own* label matches is kept even when every child was removed, which means the author sees an empty folder. That is deliberate: it tells them the folder exists and is empty of matches, rather than vanishing.

## `restore_matching`

**Contract** — for each hidden leaf whose label now matches, resolve its parent path (creating folders as needed) and reattach it, expanding to it when auto-expand is on. For each hidden folder whose label now matches, re-create it by path. Restored entries leave the hidden maps.

**Invariants** — the iteration takes a snapshot of the keys before walking, because restoring mutates the maps.

## Notes

Two behaviours fall out of the design and are worth naming, because a rebuild will have to choose them deliberately.

**A hidden node's path is frozen at detach time.** If the tree is refilled while a filter is active, the hidden maps still name the old paths, and restoring writes nodes into a tree that has moved on. Nothing prevents this; the editor avoids it by always clearing the filter before a refresh.

**Restoring a leaf re-creates its folders.** So a folder that was detached as empty comes back the moment one of its children matches, without being restored from the folder map. The folder map only matters for folders whose own label is the match.

A rebuild whose tree supports per-node visibility deletes this entire file and replaces it with a predicate evaluated at paint time — which is strictly better, and loses nothing.

**None of this runs in the shipped editor.** The text box's change notification is never bound (see [the layout file](TreeViewFilterPanel.Designer.cs.md)), and the panel is constructed hidden. The algorithm is documented here in full because it is a complete, considered design that a rebuild may want; it is documented as inert because a rebuilder comparing behaviour against the original would otherwise chase a feature that never fires.
