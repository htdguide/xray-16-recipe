# src/editors/xrSdkControls/Controls/TreeViewSearchPanel/TreeViewSearchPanel.Designer.cs

> The search bar's widgets and — unlike its filter sibling — the bindings that actually make them do something.

**Needs** — [`TreeViewSearchPanel.cs`](TreeViewSearchPanel.cs.md)
**Used by** — [`TreeViewSearchPanel.cs`](TreeViewSearchPanel.cs.md)
**Tier floor** — T4: a declarative layout description carrying five event bindings.

## Purpose

Generated layout for [`TreeViewSearchPanel`](TreeViewSearchPanel.cs.md): a label, a text box, a search button, a result counter, previous and next buttons, and a close button, arranged in two rows.

## State

```text
RECORD SearchPanelLayout
  search_label  : the word "Search:"
  search_box    : the needle; commits on the enter key, closes on escape
  search_button : commits the search, for authors who do not press enter
  result_label  : "Results: - / -" until a search runs
  prev, next    : step through matches; both start disabled
  close_button  : hides the panel and clears the tinting
  height        : two rows
```

## Notes

Five bindings live here — the text box's key handler, and one click handler per button — which is what makes this panel functional where [the filter panel](../TreeViewFilterPanel/TreeViewFilterPanel.Designer.cs.md), whose binding is missing, is not.

The search button and the enter key are the same action: the button synthesises an enter keystroke rather than calling the search directly. That is a small redundancy a rebuild collapses into one named command with two triggers.

The counter's initial text reads "Results" while the running text reads "Result", which is a cosmetic inconsistency and nothing more.
