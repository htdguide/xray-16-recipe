# src/editors/xrSdkControls/Controls/Interfaces/ITreeViewSource.cs

> Where a tree's contents come from: an object that refills the tree on demand, and knows which tree it fills.

**Needs** — [`TreeView.cs`](../TreeView/TreeView.cs.md)
**Used by** — [`TreeView.cs`](../TreeView/TreeView.cs.md)
**Tier floor** — T3: one method and one back-reference.

## Purpose

The editor's trees — the level browser, the weather-cycle browser, a tree of choices for a property — are not authored in the layout. They are filled from whatever the engine can currently see. This interface is the contract between the widget and whoever knows the contents.

## State

```text
INTERFACE TreeViewSource
  PROPERTY parent : TreeView      # set by the tree when the source is attached
  FUNCTION refresh()              # repopulate the (already cleared) tree
```

**Contract** — `refresh` is called after the tree has emptied itself, so a source appends rather than reconciles. The tree assigns itself to `parent` at attach time, so the source never has to be told where to put its nodes.

**Invariants** — a source is attached to at most one tree; attaching overwrites `parent`.

**Notes** — the clear-then-refill discipline is the same pull-not-push decision as the rest of the editor: no incremental update path exists, so a source cannot drift from its data. The cost is losing expansion and selection state on every refresh, which the tree partially works around by re-selecting the previously selected node after a sort.
