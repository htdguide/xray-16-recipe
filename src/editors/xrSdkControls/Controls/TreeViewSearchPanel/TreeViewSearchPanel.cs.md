# src/editors/xrSdkControls/Controls/TreeViewSearchPanel/TreeViewSearchPanel.cs

> Find-in-tree: tint every match, expand the path to each, and step through them one at a time.

**Needs** — [`TreeView.cs`](../TreeView/TreeView.cs.md) · [`TreeNode.cs`](../TreeNode/TreeNode.cs.md) · [`TreeViewSearchPanel.Designer.cs`](TreeViewSearchPanel.Designer.cs.md)
**Used by** — [`TreeView.Designer.cs`](../TreeView/TreeView.Designer.cs.md) · [`TreeView.cs`](../TreeView/TreeView.cs.md) · [`TreeViewCallbacks.cs`](../TreeView/TreeViewCallbacks.cs.md) · [`TreeViewSearchPanel.Designer.cs`](TreeViewSearchPanel.Designer.cs.md)
**Tier floor** — T3: substring matching and per-node colours.

## Purpose

The sibling of [the filter panel](../TreeViewFilterPanel/TreeViewFilterPanel.cs.md), and the opposite approach to the same need: where filtering removes what does not match, searching keeps the whole tree and *marks* what does. It appears on a key gesture rather than being always present, and unlike the filter, it is wired up and works.

Why both exist: filtering is for narrowing a long list you will then browse; searching is for locating one entry in a structure whose shape you want to keep seeing. For the editor's trees — a weather cycle's keyframes in time order, a level list by region — the shape usually matters.

## State

```text
RECORD SearchPanel
  tree            : TreeView
  matches         : list<TreeNode>     # in depth-first order
  cursor          : int                # which match is current; -1 when there is none
  cursor_prev_tint: colour             # what the current match looked like before it became current
```

**Invariants** — `cursor` indexes `matches` whenever the step buttons are enabled, and the step buttons are enabled exactly when there is a match on that side. The current match wears a distinct tint; all other matches wear the match tint; everything else wears the tree's own colours.

## Searching

**Contract** — committing the search box clears the previous results, walks the whole tree depth-first, and for each node: resets its background, collapses it, and if its label contains the search text (case-insensitively), tints it, records it, and expands every ancestor so it is reachable. Afterwards the tree's focus is cleared, and the first match becomes the current one.

```text
ON search_committed(text)
  matches = empty
  needle = lowercase(text)
  IF needle is not empty THEN walk(tree.nodes, needle)
  clear tree focus
  IF matches is empty          THEN report "no matches" and stop
  IF matches has exactly one   THEN report "1 / 1"
  ELSE enable stepping ; cursor = 0 ; report "1 / n"
       scroll match 0 into view ; make it the current match

FUNCTION walk(nodes, needle)
  FOR EACH node IN nodes
    node.background = the tree's ordinary background
    collapse node
    IF needle IN lowercase(node.label)
      node.background = the match tint
      append node to matches
      expand every ancestor of node
    walk(node.children, needle)
```

**Invariants** — the collapse happens *before* the match test on that node but the ancestor expansion happens after, and the walk is pre-order — so a folder that is collapsed early is re-expanded when one of its descendants matches. That ordering is what produces the desired end state: everything collapsed except the paths to matches.

**Notes** — the single-match case enables no stepping and does not scroll to or highlight the one match, so a search with exactly one result reports it and shows nothing. That is a defect, not a decision.

## Stepping

**Contract** — next and previous move the cursor by one, restore the outgoing match's tint, remember and replace the incoming one's, scroll it into view, clear the tree's focus, and update the counter. A button disables itself when the cursor reaches its end, and enables its opposite.

**Invariants** — the cursor is moved *before* being used, and there is no bounds check — the buttons' enabled state is the only guard. That works because each step disables the button that would run off the end, but it means the invariant "a step button is enabled only when a step exists" is load-bearing, not merely cosmetic.

**Notes** — the outgoing match's tint is restored from a saved colour rather than recomputed, which is why the panel stores one. It has to: a match that is also selected wears the selection colours, and recomputing would lose that.

## `close`

**Contract** — hides the panel, restores every node in the tree to the tree's own foreground and background, disables stepping, forgets the matches and resets the counter. Bound to the close button and to the escape key.

**Invariants** — the restore walks the *whole* tree, not just the matches, because a previous search may have tinted nodes that this one did not clear.

**Notes** — restoring to the tree's colours also erases the multiple-selection painting done by [`TreeViewSelection`](../TreeView/TreeViewSelection.cs.md), so closing the search panel visually deselects a selection that is still logically there. The two features both own a node's colour and neither knows about the other. A rebuild makes selection and match-tint two render-time states composited at paint, rather than two writers of one stored colour.

## Notes

Matching is a case-insensitive substring test against the node's own label, not its path — so searching for a folder name does not find its contents, and searching for a full path finds nothing. That is a reasonable default for a find box and worth stating because the tree is otherwise addressed by path everywhere.
