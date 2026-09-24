# src/editors/xrSdkControls/Controls/TreeNode/TreeNode.cs

> A tree entry that knows whether it is a leaf or a folder, and which icon it wears when open and when closed.

**Needs** — [`TreeView.cs`](../TreeView/TreeView.cs.md)
**Used by** — [`TreeView.cs`](../TreeView/TreeView.cs.md) · [`TreeViewCallbacks.cs`](../TreeView/TreeViewCallbacks.cs.md) · [`TreeViewSelection.cs`](../TreeView/TreeViewSelection.cs.md) · [`TreeViewFilterPanel.cs`](../TreeViewFilterPanel/TreeViewFilterPanel.cs.md) · [`TreeViewSearchPanel.cs`](../TreeViewSearchPanel/TreeViewSearchPanel.cs.md)
**Tier floor** — T3: a record with two constructors' worth of convenience.

## Purpose

The editor's trees are built from paths — `weathers/default/06:00`, `levels/l01_escape`. Every path segment but the last is a folder, and the tree behaves differently for the two: folders are usually not selectable, they collapse, and they change icon when they do. The stock node type carries none of that, so this one adds it.

## State

```text
RECORD TreeNode
  kind                   : ENUM { leaf, folder }
  name                   : text        # the path segment; also the lookup key among siblings
  selected               : bool        # this tree paints its own selection, see TreeViewSelection
  icon_expanded          : optional<int>
  icon_collapsed         : optional<int>
  children               : list<TreeNode>
```

**Invariants** — the display text and the lookup key are the same string, set together at construction. That is what lets [`TreeView`](../TreeView/TreeView.cs.md) resolve a path by walking segment-by-segment through sibling lookups; if they could differ, path resolution would silently miss.

**Notes** — the node carries its own `selected` flag, shadowing the host toolkit's. That is not redundancy: the tree implements multiple selection itself by colouring nodes (see [`TreeViewSelection`](../TreeView/TreeViewSelection.cs.md)), and the toolkit's own flag only ever describes the single focused node.

The two icon slots are indices into an image list the tree owns, with "absent" meaning "use the default for this kind". Storing absent rather than a default index lets a node inherit a change to the defaults.

## `add_leaf` / `add_folder`

**Contract** — create a child of the given kind under this node, inheriting this node's context menu, and return it. An icon may be supplied; without one, the node takes the kind's default index — a fixed index for folders, a different fixed index for leaves.

**Invariants** — the child is appended, not inserted in order; ordering is the tree's job, done by a sort after a batch of additions.

**Notes** — `add_folder` sets the new node's kind to *leaf*. It records folder icons, is named for folders, and is used for folders — and then marks the node a leaf, which makes it selectable in a tree configured to refuse folder selection. This is a bug, not a decision; a rebuild sets the kind to folder.

## Notes

The default icon indices are bare numbers in this file — one for a closed folder, one for an open folder, one for a leaf — and their meaning lives in whichever image list the host tree was configured with. Nothing names them. A rebuild should name them.
