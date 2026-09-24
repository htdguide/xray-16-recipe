# src/editors/xrSdkControls/Controls/TreeViewSearchPanel

## What this module is responsible for

Finding entries in a tree without changing its shape: tint every match, collapse everything else, expand the path to each match, and step through them one at a time with a running count.

## Where it sits and what it rests on

It rests on [`TreeView`](../TreeView/README.md) and [`TreeNode`](../TreeNode/README.md). [`TreeView`](../TreeView/README.md) constructs one for every tree and opens it on a key gesture.

## The load-bearing ideas

**Search marks, filter removes.** These are the two halves of the same need and the editor ships both: filtering is for narrowing a long list you will then browse; searching is for locating one entry in a structure whose shape you want to keep seeing. For a weather cycle's keyframes in time order, the shape matters, which is why this is the one that is wired up.

**Collapse before the match test, expand after.** The walk is pre-order and collapses each node as it reaches it, then expands the ancestors of every match. The end state is everything collapsed except the paths to matches, and it falls out of the ordering rather than needing a second pass.

**The current match's previous tint is saved, not recomputed.** A match that is also selected wears the selection colours; recomputing would lose them.

## The twins

| File | Role |
|---|---|
| [`TreeViewSearchPanel.cs`](TreeViewSearchPanel.cs.md) | The search walk, the tinting, the stepping, and the close-and-restore |
| [`TreeViewSearchPanel.Designer.cs`](TreeViewSearchPanel.Designer.cs.md) | The two-row bar and the five bindings that make it work |

A resource file accompanies the layout; it holds no decision and has no twin.

## What the twins record as defects

A search with exactly one result reports it and then neither scrolls to it nor highlights it. Stepping has no bounds check and relies entirely on the buttons' enabled state as its guard. And closing the panel repaints the whole tree in its default colours, which erases [the tree's own selection painting](../TreeView/TreeViewSelection.cs.md) — the two features both own a node's colour and neither knows about the other.
