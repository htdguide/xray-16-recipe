# src/editors/xrSdkControls/Controls/NumericSpinner/NumericSpinner.Designer.cs

> Layout for the drag-scrub spinner: the number on the left, a narrow grip on the right, with the three drag gestures bound to it.

**Needs** — [`NumericSpinner.cs`](NumericSpinner.cs.md)
**Used by** — [`NumericSpinner.cs`](NumericSpinner.cs.md)
**Tier floor** — T4: a declarative layout description that also names three event bindings.

## Purpose

Generated layout for [`NumericSpinner`](NumericSpinner.cs.md). Unlike the other layout files in this directory it carries one behavioural fact: the press, move and release of the grip are wired here, so a rebuild reading only the behaviour file would not know where the gesture is anchored.

## State

```text
RECORD NumericSpinnerLayout
  spinner : the host toolkit's fractional spin box, fills the left
  handle  : a narrow button at the right, labelled with a horizontal double arrow
  height  : one text row plus a pixel of border
```

## Notes

The grip is labelled with a text glyph rather than an icon, which is why the control has no image resource. The label is what tells an author the gesture exists at all, so a rebuild must keep some affordance there — a bare button would be undiscoverable.
