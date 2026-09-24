# src/editors/xrSdkControls/Controls/TreeView/TreeViewSelection.cs

> Multiple selection as a painting problem: the tree keeps the list, and a selected node is one that has been recoloured.

**Needs** — [`TreeView.cs`](TreeView.cs.md) · [`TreeNode.cs`](../TreeNode/TreeNode.cs.md)
**Used by** — [`TreeView.cs`](TreeView.cs.md) · [`TreeViewCallbacks.cs`](TreeViewCallbacks.cs.md)
**Tier floor** — T3: list maintenance and per-node colours.

## Purpose

The host toolkit's tree has exactly one selected node. The editor needs many — deleting a batch of keyframes, retargeting a group of levels. Rather than replace the widget, [`TreeView`](TreeView.cs.md) keeps its own selection list and *paints* it, by giving each selected node the system highlight colours. This file is that mechanism.

## State

Borrowed from [`TreeView`](TreeView.cs.md): `selected : list<TreeNode>`, and each node's own `selected` flag.

**Invariants** — a node's flag and its membership in the list are always consistent; both are set together and cleared together. Both exist because the flag makes "is this node already selected" a constant-time question during a range walk over hundreds of nodes, while the list is what the host reads.

## `select` / `deselect`

**Contract** — selecting sets the node's flag, paints it with the system highlight foreground and background, appends it to the list, and invalidates just that node's rectangle. Deselecting clears the flag, restores the *unset* colours (so the node inherits the tree's, rather than a remembered pair), removes it from the list, and invalidates the same rectangle. Selecting an already-selected node, or deselecting a non-node, does nothing.

**Notes** — restoring "unset" rather than a saved original is the decision, and it is right here because nothing else colours a node — with one exception: [the search panel](../TreeViewSearchPanel/TreeViewSearchPanel.cs.md) tints matches, and a select-then-deselect over a search result loses the tint. A rebuild that keeps both features must decide which owns a node's colour; the honest answer is neither, and selection should be a render-time state rather than a stored colour.

Invalidating only the node's own rectangle rather than the whole tree is what keeps a shift-select over a long range from flickering.

## `select_all`

**Contract** — walks the visible-node chain from the first node to the end, selecting each node that may be selected. Folders are skipped when the tree forbids selecting them.

**Notes** — visible, not all: a collapsed folder's contents are not included. See [`TreeViewCallbacks`](TreeViewCallbacks.cs.md).

## `deselect_all`, `deselect_many`

**Contract** — deselect every node in the tree's selection, or in a given list.

**Notes** — `deselect_all` iterates the selection list while `deselect` removes from it. Whether that is safe depends entirely on the iteration primitive the host happens to provide, and it is the kind of thing that works until the container is changed underneath it. A rebuild copies the list, or clears it and repaints, rather than mutating during a walk.

## `select_subtree`

**Contract** — selects a node's descendants recursively. Present and unreferenced: nothing in the tree offers a "select this folder's contents" gesture. It is either a missing feature or dead code; the source does not say which.
