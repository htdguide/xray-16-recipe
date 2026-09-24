# src/editors/xrSdkControls/Controls/TreeNode

## What this module is responsible for

One entry in an editor tree, extended with the three things the host toolkit's own node does not carry: whether it is a leaf or a folder, which icon it wears open and closed, and its own selection flag.

## Where it sits and what it rests on

It rests on the host toolkit's node type. [`TreeView`](../TreeView/README.md) and both attached panels are built on it.

## The load-bearing ideas

**The display text and the lookup key are the same string.** That is what lets the tree resolve a slash-separated path by walking segment by segment through sibling lookups. If the two could differ, path resolution would silently miss.

**Leaf or folder is a stored kind, not an inference from having children.** A folder with no children is still a folder, which matters to filtering (an empty matching folder is shown) and to selection (folders may be unselectable).

**Absent means "use the default", not "use index zero".** The two icon slots store absence explicitly so a node inherits a later change to the defaults.

## The twins

| File | Role |
|---|---|
| [`TreeNode.cs`](TreeNode.cs.md) | The node: kind, key, icons, own selection flag, and two typed child-adding helpers |

## What the twins record as defects

The folder-adding helper marks the node it creates a *leaf*, which makes folders selectable in trees configured to refuse that. The default icon indices are bare numbers whose meaning lives in whichever image list the tree was given, and nothing names them.
