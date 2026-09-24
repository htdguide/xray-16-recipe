# src/editors/xrSdkControls/Controls/TreeView/TreeView.cs

> A tree addressed by path rather than by node: give it `a/b/c` and the intermediate folders come into existence.

**Needs** — [`TreeNode.cs`](../TreeNode/TreeNode.cs.md) · [`ITreeViewSource.cs`](../Interfaces/ITreeViewSource.cs.md) · [`TreeViewFilterPanel.cs`](../TreeViewFilterPanel/TreeViewFilterPanel.cs.md) · [`TreeViewSearchPanel.cs`](../TreeViewSearchPanel/TreeViewSearchPanel.cs.md) · [`TreeView.Designer.cs`](TreeView.Designer.cs.md) · [`TreeViewCallbacks.cs`](TreeViewCallbacks.cs.md) · [`TreeViewSelection.cs`](TreeViewSelection.cs.md)
**Used by** — [`ITreeViewSource.cs`](../Interfaces/ITreeViewSource.cs.md) · [`TreeNode.cs`](../TreeNode/TreeNode.cs.md) · [`TreeView.Designer.cs`](TreeView.Designer.cs.md) · [`TreeViewCallbacks.cs`](TreeViewCallbacks.cs.md) · [`TreeViewSelection.cs`](TreeViewSelection.cs.md) · [`TreeViewFilterPanel.cs`](../TreeViewFilterPanel/TreeViewFilterPanel.cs.md) · [`TreeViewSearchPanel.cs`](../TreeViewSearchPanel/TreeViewSearchPanel.cs.md)
**Tier floor** — T3: path splitting, node bookkeeping and a container swap.

## Purpose

Everything the editor browses is named by a slash-separated logical path — levels, weather cycles and their keyframes, textures, particle effects. The tree that shows them therefore speaks paths, not nodes. This is the core of that tree: the path walk, the item and group operations built on it, and the wrapper panel that gives every tree a filter box and a search box without the caller arranging them.

The class is split across four files by concern — this one is structure and the public surface, [callbacks](TreeViewCallbacks.cs.md) is input, [selection](TreeViewSelection.cs.md) is multi-select painting, and [the designer file](TreeView.Designer.cs.md) is the container and context menu. The split is arbitrary in the sense that a rebuild may merge them; it is useful in that the selection logic is genuinely independent.

## State

```text
RECORD TreeView
  root              : TreeNode              # the invisible node above the top level
  source            : optional<TreeViewSource>   # who refills this tree
  context_menu      : the menu offered on a node
  selected          : list<TreeNode>        # the tree's own multi-selection
  last_selected     : optional<TreeNode>    # the anchor for a range extension
  just_selected     : optional<TreeNode>    # guards a re-entrant select, see callbacks
  multi_select      : bool
  selectable_groups : bool                  # may a folder be selected at all
  container         : Panel                 # holds the filter panel, the search panel and the tree
  container_built   : bool                  # the swap below happens exactly once
```

**Invariants** — a node's key among its siblings is its path segment, which is how the walk below can resolve a path without keeping an index. The tree never holds its data: it is cleared and refilled from its source.

## `resolve_path` — the operation everything else is built on

**Contract** — walks a sequence of path segments from the root, returning the node it reaches. In *creating* mode a missing segment is made as a folder and the walk continues; in *non-creating* mode a missing segment aborts the walk and reports nothing. Empty segments are skipped, so a doubled or trailing separator is harmless.

```text
FUNCTION resolve_path(segments, create : bool, icons) -> optional<TreeNode>
  node = child of root named segments[0], creating a folder if absent
  FOR EACH segment IN segments AFTER THE FIRST
    IF segment is empty THEN CONTINUE
    IF node has a child keyed segment
      node = that child ; CONTINUE
    IF NOT create THEN RETURN none
    node = new folder child of node, keyed and labelled segment, with the given icons
  RETURN node
```

**Notes** — the first segment is created unconditionally even in non-creating mode. That asymmetry is not justified anywhere; it means "does `missing/thing` exist" leaves a stray empty top-level folder behind. A rebuild should make the whole walk uniform.

## `add_item`

**Contract** — inserts a leaf at a path: the leading segments resolve (creating folders), the final segment becomes the leaf. Re-sorts the tree afterwards and restores the previously selected node, so building a tree item by item does not scroll the author's selection away.

**Invariants** — when an explicit icon is supplied, the folder walk is done in *non-creating* mode, so an item with a custom icon lands at the top level if its folders do not already exist; without an icon, folders are created. Nothing states why the two differ, and it reads like an accident rather than a rule.

**Notes** — sorting after *every* single insertion makes filling a tree of n items cost n sorts. For the tens-to-hundreds of entries these trees hold it does not matter, and saying that out loud is the point: it is a known, accepted cost, not an oversight to preserve.

## `add_group`, `remove_item`, `remove_group`

**Contract** — `add_group` is the creating path walk, returning the folder at the end. `remove_item` is the non-creating walk followed by detaching whatever it found. `remove_group` is the same call under a second name — folders and leaves are removed identically, and the two names exist only to read correctly at the call site.

## `find`, `select_path`

**Contract** — `find` resolves a slash-separated path against the existing tree, starting from the root node, and returns what it reaches, or nothing if it never left the root. `select_path` finds and selects, doing nothing if the path is absent.

**Notes** — two things are wrong here and both are worth recording so a rebuild does not reproduce them.

The walk *ignores* a missing segment and keeps going rather than failing, so a path whose middle is wrong resolves to whatever its correct prefix reached: `weathers/typo/06:00` returns the `weathers` folder. A rebuild fails the walk on the first missing segment.

And the root it starts from is never obtained. The tree tries to reach the underlying widget's own hidden root node by name at construction, and looks in the wrong visibility — the name it searches for is its *own* public slot, so the lookup yields nothing and the slot stays empty. Every call to `find` therefore walks from nothing. A rebuild keeps an explicit root of its own instead of borrowing the widget's.

## `refresh`

**Contract** — empties the tree, asks the source to refill it, then repaints. The source is optional; without one the tree simply empties.

**Invariants** — clear-then-refill, never incremental. Expansion and selection state are lost. That is the pull-not-push decision the whole editor makes: there is no reconciliation path, so the tree cannot drift from its data.

## `Source`

**Contract** — attaching a source stores it and tells it which tree it fills, so the source never has to be told twice.

## Notifications

**Contract** — the tree announces five things to its host: the author asked to create an item, to create a group, or to remove the selection (all three raised from the context menu); the source finished loading; and the selection changed. The first three are *requests* — the tree does not act on them, because what "create an item" means is the host's business.

## Container substitution

**Contract** — the first time the tree is given a parent, it replaces itself in that parent with a panel of its own, at the same position, size and docking, and then puts itself inside that panel, filling what the filter and search panels (docked along one edge, both hidden until asked for) leave.

**Invariants** — happens exactly once, guarded by a flag, because a control may be re-parented later. The tree preserves its index among its siblings across the swap so tab order and z-order survive.

**Notes** — this is a widget-toolkit trick with a real decision underneath: **a caller drops one tree into a layout and silently gets a tree plus a filter box plus a search box**, without arranging three controls and wiring them together. A rebuild expresses this as a composite widget that contains a tree rather than a tree that smuggles in a container — cleaner, and it removes the one-shot flag entirely.
