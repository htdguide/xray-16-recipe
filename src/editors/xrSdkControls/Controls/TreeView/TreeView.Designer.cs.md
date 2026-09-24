# src/editors/xrSdkControls/Controls/TreeView/TreeView.Designer.cs

> The tree's defaults, its wrapper panel, the two attached panels, and the three-item node menu every tree gets for free.

**Needs** — [`TreeView.cs`](TreeView.cs.md) · [`TreeViewFilterPanel.cs`](../TreeViewFilterPanel/TreeViewFilterPanel.cs.md) · [`TreeViewSearchPanel.cs`](../TreeViewSearchPanel/TreeViewSearchPanel.cs.md)
**Used by** — [`TreeView.cs`](TreeView.cs.md)
**Tier floor** — T4: a declarative construction description, though it does carry three real defaults.

## Purpose

Generated construction for [`TreeView`](TreeView.cs.md). Unlike the other layout files in this library it is not purely cosmetic: it sets the tree's behavioural defaults and it is where the filter and search panels are attached and told which tree they act on.

## State

```text
RECORD TreeViewConstruction
  multi_select      = true       # every tree allows multiple selection unless told otherwise
  selectable_groups = true       # folders are selectable unless told otherwise
  path_separator    = "/"        # the separator the path walk splits on
  container         : a panel holding the filter panel, the search panel and the tree
  filter_panel      : docked to the bottom of the container, hidden
  search_panel      : docked to the bottom of the container, hidden
  context_menu      : three items — add folder, add item, remove item
```

## Notes

**The path separator is the load-bearing default.** It must match the separator used by every caller that addresses this tree by path, and it matches the engine's own logical resource paths, which are forward-slash separated. A rebuild that changes it breaks every call site silently, because a path with the wrong separator resolves to a single node named for the whole path.

Both attached panels are constructed hidden and docked to the *bottom* of the wrapper. Bottom rather than top is a presentation choice; hidden is not — the search panel appears on a key gesture and the filter panel is shown by the host through the tree's own visibility property.

The menu items are bound to the tree's three *request* notifications rather than to actions. That indirection is the point: this library does not know what an item is, so choosing "add item" tells the host that the author asked, and the host — which is browsing levels, or weather cycles, or particle effects — decides what to make.

The menu is stored on the tree, but it is attached to *nodes* as they are created, not to the tree itself. So a right-click on empty space offers nothing, which is consistent with the menu's items all being about a node.
