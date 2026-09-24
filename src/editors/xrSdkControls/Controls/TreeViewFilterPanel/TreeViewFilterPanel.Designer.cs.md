# src/editors/xrSdkControls/Controls/TreeViewFilterPanel/TreeViewFilterPanel.Designer.cs

> A label and a text box, one row high — and the place where the filter's change notification should have been bound, but is not.

**Needs** — [`TreeViewFilterPanel.cs`](TreeViewFilterPanel.cs.md)
**Used by** — [`TreeViewFilterPanel.cs`](TreeViewFilterPanel.cs.md)
**Tier floor** — T4: a declarative layout description.

## Purpose

Generated layout for [`TreeViewFilterPanel`](TreeViewFilterPanel.cs.md). Two widgets and a height.

## State

```text
RECORD FilterPanelLayout
  label    : the word "Filter:"
  text_box : fills the remaining width, stretches with the panel
  height   : one text row plus padding; fixed
```

## Notes

**The text box's change notification is never bound to anything.** The filtering algorithm in [`TreeViewFilterPanel`](TreeViewFilterPanel.cs.md) is complete and correct and nothing calls it — this file is where the binding belongs, and it is absent. The feature is inert in the shipped editor.

A rebuild should bind it. The algorithm it would drive is documented in full in the behaviour file, and the binding is one line; whether it was removed deliberately (the detach-and-restore approach is fragile against a tree refresh) or lost is not recoverable from the source.
