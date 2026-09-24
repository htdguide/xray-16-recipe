# src/editors/xrSdkControls/Controls/TreeView/TreeViewCallbacks.cs

> Every input gesture the tree answers: the keyboard shortcuts, the right-click rule, the icon swap on expand, and the multi-selection state machine.

**Needs** — [`TreeView.cs`](TreeView.cs.md) · [`TreeViewSelection.cs`](TreeViewSelection.cs.md) · [`TreeNode.cs`](../TreeNode/TreeNode.cs.md) · [`TreeViewSearchPanel.cs`](../TreeViewSearchPanel/TreeViewSearchPanel.cs.md) · [Seam: Windowing and input](../../../../../SYSTEM-REQUIREMENTS.md#seam-windowing-and-input)
**Used by** — [`TreeView.cs`](TreeView.cs.md)
**Tier floor** — T3: input interpretation.

## Purpose

The input half of [`TreeView`](TreeView.cs.md). Three of its five concerns are small; the fifth — reconciling a toolkit that has one selected node with a tree that wants many — is the substance of the file and the reason it exists separately.

## State

Borrowed from [`TreeView`](TreeView.cs.md): the selection list, the range anchor, the re-entrancy guard, and the right-click flag.

## Keyboard

**Contract** — with the modifier key held: **F** opens the search panel, clears it, focuses it and disables the step buttons; **A** selects every visible node, when multiple selection is enabled; **D** deselects everything. All other keys fall through to the tree's default handling.

**Notes** — select-all walks *visible* nodes, so a collapsed folder's contents are not selected. That is the right choice for a gesture whose feedback is visual: the author selects what they can see.

## Right-click

**Contract** — a right press over a node that is already selected raises a flag; the flag suppresses the selection change that a right-click would otherwise cause, and is cleared on release. A right-click on a node that is *not* selected selects it first.

**Invariants** — the flag is checked at the start of every selection attempt, so the rule is exactly: **right-clicking inside an existing selection opens the menu on that whole selection; right-clicking outside it replaces the selection with the one node**. That is what makes "remove" meaningful on a multi-selection.

## Expand and collapse

**Contract** — on collapse the node takes its collapsed icon; on expand, its expanded icon. A node with no icons takes a fixed default for each state.

**Notes** — the expand branch tests the *collapsed* icon slot and then uses the *expanded* one. A node with an expanded icon but no collapsed icon therefore shows a default on expand instead of its own. A rebuild tests the slot it is about to use.

## Selection

**Contract** — this is a state machine over three inputs (the modifier held, whether the node is already selected, whether the node is a folder) and it implements the ordinary three-gesture selection vocabulary:

```text
ON before_select(node)
  IF right_click_in_selection THEN cancel
  IF folders are not selectable AND node is a folder THEN cancel
  IF multi_select AND control_held AND node is already selected
    IF node is the one we just selected THEN cancel silently   # re-entrancy guard
    deselect(node) ; clear focus ; cancel ; announce selection changed

ON after_select(node)
  IF NOT multi_select
    selection = { node } ; announce ; RETURN
  just_selected = node ; clear focus        # the toolkit's own focus is not the selection
  IF control_held AND node not selected
    select(node) ; anchor = node ; announce ; RETURN
  IF NOT shift_held
    deselect all ; select(node) ; anchor = node ; announce ; RETURN
  # shift: extend from the anchor to this node along the visible order
  walk visible nodes from the lower of (anchor, node) upward to the higher,
    selecting each one that is selectable
  anchor = node ; announce
```

**Invariants** — after every gesture the tree's own focused-node slot is cleared, because the toolkit would paint that one node with its own highlight and the tree paints selection itself. The `just_selected` guard exists because clearing focus re-enters the selection handler; without it, control-clicking a selected node would deselect and immediately reselect it.

**Notes** — the range walk orders nodes by their **vertical screen position**, not by tree structure. That is correct for the gesture (shift-click selects what looks like a range on screen) and it is why the walk uses the visible-node chain rather than recursion. It also means the result depends on what is expanded, which is what the author expects.

The range walk starts from the lower node and steps *upward*, and selects the final node outside the loop — because the loop stops at the upper node rather than past it. The structure is awkward but the boundary is right: both ends are selected.

## Re-parenting

**Contract** — the first time the tree is given a parent, the container substitution described in [`TreeView`](TreeView.cs.md) happens here. Guarded to run once.
