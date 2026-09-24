# src/editors/xrSdkControls/Controls/PropertyGrid.cs

> The property grid, extended with three gestures the stock grid has no vocabulary for: drag-to-scrub, double-click-to-open, and a remembered splitter position.

**Needs** — [`IProperty.cs`](Interfaces/IProperty.cs.md) · [`IPropertyContainer.cs`](Interfaces/IPropertyContainer.cs.md) · [`IIncrementable.cs`](Interfaces/IIncrementable.cs.md) · [`IMouseListener.cs`](Interfaces/IMouseListener.cs.md) · [Seam: Windowing and input](../../../../SYSTEM-REQUIREMENTS.md#seam-windowing-and-input)
**Used by** — [`IMouseListener.cs`](Interfaces/IMouseListener.cs.md)
**Tier floor** — T3: it reaches into a widget it did not write to find a private layout field, which a rebuild would not have to do.

## Purpose

Authoring weather means nudging a hundred numbers and watching the sky change. A grid that only accepts typed values makes that unbearable. This subclass adds the two gestures that make it bearable — middle-drag over a row to scrub its value continuously, and double-click a row to open its own editor — and remembers the label/value splitter between sessions.

The load-bearing idea is **the routing**, not the gestures: the grid receives a mouse event on an anonymous row, and must get from there to the engine-side object that owns the value. It does that by asking the row's descriptor for its container, asking the container for the property behind this row, and then asking whether that property claims one of two optional capabilities.

## State

```text
RECORD PropertyGrid
  view          : the inner row-rendering widget, found by name at construction
  prev_location : the last mouse position seen, used to derive a drag delta
```

**Invariants** — every gesture handler returns immediately unless there is both a selected object and a selected row. The capability test is a query, not a cast that must succeed: a property that implements neither optional interface simply ignores the gesture.

## Gesture routing

```text
FUNCTION property_under_cursor() -> optional<Property>
  IF no selected object OR no selected row THEN RETURN none
  descriptor = selected_row.descriptor
  container  = descriptor.owning_container          # a PropertyContainer, or not
  IF container IS NOT a PropertyContainer THEN RETURN none
  RETURN container.property_for(descriptor.spec)
```

**Notes** — this walk exists because the grid widget and the property model were written by different people: the grid hands back a *descriptor*, the model is keyed by *specification*. A rebuild that owns both ends stores the property on the row and deletes the walk.

## Middle-drag scrubbing

**Contract** — while the middle button is held, each horizontal mouse movement asks the property under the cursor to increment itself by the horizontal delta in pixels, then forces a repaint so the new value is visible immediately. A property that does not declare itself incrementable is skipped.

**Invariants** — the delta is the difference from the previous position, and the previous position is updated on every motion event *including* those with no button held, so releasing and re-pressing the button does not produce one huge jump.

```text
ON mouse_move(position, buttons)
  IF middle NOT IN buttons
    prev_location = position ; RETURN     # keep the anchor fresh while idle
  IF position.x == prev_location.x THEN RETURN
  property = property_under_cursor()
  IF property does not support increment THEN RETURN
  property.increment(position.x - prev_location.x)
  repaint()
  prev_location = position
```

**Notes** — the increment is given in *pixels*, and each property decides what a pixel is worth in its own units. That is the right split: the grid knows about the gesture, the property knows about the quantity.

## Double-click

**Contract** — forwards a double-click to the property under the cursor if that property declares itself a mouse listener, passing the grid itself so the property can open a dialog parented correctly and refresh the grid afterwards.

## `save` / `load`

**Contract** — persist and restore the width of the column splitter under a named key in the platform's per-user settings store. Restoring a missing key leaves the current width.

**Notes** — the splitter width lives in a private field of the widget the grid did not write, and is reached by name through reflection. That is the incidental part; the decision is that **the splitter position is per-user state worth surviving a restart**, because the weather grid's identifiers are long and the default split truncates them. A rebuild whose grid exposes the splitter writes two lines here.
