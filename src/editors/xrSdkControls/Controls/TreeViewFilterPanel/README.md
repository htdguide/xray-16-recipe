# src/editors/xrSdkControls/Controls/TreeViewFilterPanel

## What this module is responsible for

Narrowing a tree as the author types: entries that do not match are removed from the tree and remembered, and a shortened filter puts them back.

## Where it sits and what it rests on

It rests on [`TreeView`](../TreeView/README.md)'s path walk, which is what restores a removed node to the right place. [`TreeView`](../TreeView/README.md) constructs one for every tree.

## The load-bearing ideas

**Filtering detaches, it does not hide.** The host toolkit's tree has no per-node visibility, so a node that must not be shown must leave the tree. Everything else in the design follows from that: the panel must remember what it removed, and it remembers by *path*, because the path is what the tree's own walk can resolve.

**Folders and leaves are remembered differently.** A detached leaf is kept by object; a detached folder is kept by label only, because its children already went into the leaf map under their own full paths. Restoring a leaf re-creates its folders on the way.

**Removal runs before restoration.** A node that fails the new filter must leave before one that passes it arrives, or the sort observes a state that was never meant to exist.

**A folder whose own name matches survives empty.** The author sees that the folder exists and contains no matches, rather than watching it vanish.

## The twins

| File | Role |
|---|---|
| [`TreeViewFilterPanel.cs`](TreeViewFilterPanel.cs.md) | The detach-and-restore algorithm, and the two maps that remember what left |
| [`TreeViewFilterPanel.Designer.cs`](TreeViewFilterPanel.Designer.cs.md) | A label and a text box, one row high — and the missing binding |

A resource file accompanies the layout; it holds no decision and has no twin.

## What the twins record as defects

**The feature is inert.** The text box's change notification is never bound to the filtering code, and the panel is constructed hidden. The algorithm is documented in full because it is a complete design a rebuild may want; it is documented as dead so nobody chases behaviour that never fires.

Separately, a hidden node's remembered path is frozen at detach time, so refreshing the tree with a filter active restores nodes into a tree that has moved on.
