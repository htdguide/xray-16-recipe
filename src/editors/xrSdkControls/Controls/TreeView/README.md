# src/editors/xrSdkControls/Controls/TreeView

## What this module is responsible for

The editor's tree widget: addressed by path rather than by node, painting its own multiple selection, and silently bringing a filter box and a search box with it wherever it is dropped.

## Where it sits and what it rests on

It rests on [`TreeNode`](../TreeNode/README.md), on [`ITreeViewSource`](../Interfaces/ITreeViewSource.cs.md) for its contents, and on [`TreeViewFilterPanel`](../TreeViewFilterPanel/README.md) and [`TreeViewSearchPanel`](../TreeViewSearchPanel/README.md) for the two attached panels. Every browsing panel in the weather editor — levels, weather cycles, the tree-of-choices dialog — is one of these.

The class is split across four files. The split between structure, input and selection is useful; a rebuild may merge them freely.

## The load-bearing ideas

**Paths, not nodes.** Everything the editor browses is named by a slash-separated logical path, so the tree's primitive operation is a path walk that creates missing folders as it goes. Adding an item at `a/b/c` brings `a` and `b` into existence. The path separator is a single configured character and must match every caller's.

**Multiple selection is painted, not delegated.** The host toolkit's tree selects one node; the editor needs many. So the tree keeps its own list, colours its members, and clears the toolkit's own focus after every gesture so two highlights never fight. Each node also carries a selection flag, so "is this already selected" is a constant-time question during a range walk.

**Range selection follows screen order, not tree structure.** Shift-clicking selects what looks like a range, which depends on what is expanded — and is what the author expects.

**Right-clicking inside a selection keeps it.** Outside it, the click replaces the selection with the one node. This is what makes a "remove" menu item meaningful on a multi-selection.

**Refresh is clear-then-refill.** There is no incremental reconciliation anywhere, so the tree cannot drift from its data; the cost is losing expansion and selection state.

**The tree smuggles in its own container.** The first time it is given a parent it replaces itself with a panel and re-enters that panel below the filter and search bars — so a caller drops in one widget and gets three. A rebuild expresses this as a composite widget containing a tree, which is cleaner and deletes the one-shot guard.

## The twins

| File | Role |
|---|---|
| [`TreeView.cs`](TreeView.cs.md) | The path walk and everything built on it: add, remove, find, refresh, the source binding, and the container substitution |
| [`TreeViewCallbacks.cs`](TreeViewCallbacks.cs.md) | Input: the keyboard shortcuts, the right-click rule, the icon swap on expand, and the selection state machine |
| [`TreeViewSelection.cs`](TreeViewSelection.cs.md) | Multiple selection as painting: select, deselect, select-all, and the node/list consistency rule |
| [`TreeView.Designer.cs`](TreeView.Designer.cs.md) | Defaults (multi-select on, folders selectable, `/` as separator), the wrapper panel, and the three-item node menu |

## What the twins record as defects

The root node the path lookup walks from is fetched by name in the wrong visibility and is always absent, so `find` walks from nothing. The path walk ignores a missing middle segment instead of failing, so a wrong path resolves to its correct prefix. The non-creating walk still creates its first segment. The expand handler tests the collapsed icon slot and uses the expanded one. Each is named in the twin that carries it.
