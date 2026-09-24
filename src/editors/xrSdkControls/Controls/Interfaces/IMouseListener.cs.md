# src/editors/xrSdkControls/Controls/Interfaces/IMouseListener.cs

> An optional capability a property may claim: "a double-click on me opens something".

**Needs** — [`PropertyGrid.cs`](../PropertyGrid.cs.md)
**Used by** — [`PropertyGrid.cs`](../PropertyGrid.cs.md) · [`property_color_base.cpp`](../../../xrWeatherEditor/property_color_base.cpp.md) · [`property_color_base.hpp`](../../../xrWeatherEditor/property_color_base.hpp.md)
**Tier floor** — T3: one method.

## Purpose

Some property types answer a double-click with a modal of their own — a colour dialog, a file browser, a tree of choices. The grid cannot know which, so it asks. A property that does not claim this capability gets the grid's default behaviour (cycle the value, or begin inline editing).

## State

`Stateless.`

## `IMouseListener`

```text
INTERFACE MouseListener
  FUNCTION on_double_click(grid : PropertyGrid)
```

**Contract** — the grid passes itself so the property can parent its dialog correctly and force a repaint after the author commits. The call blocks for as long as the property's dialog is open, which means **the engine's idle pump is starved while a property dialog is up** — the rendered view freezes. That is a consequence of the editor owning the loop, and it is visible to the author.

**Notes** — a rebuild that inverts control (engine owns the loop, editor is a panel) does not have this problem, because the modal is drawn inside the frame rather than instead of it.
