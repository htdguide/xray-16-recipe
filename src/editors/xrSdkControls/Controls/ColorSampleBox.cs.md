# src/editors/xrSdkControls/Controls/ColorSampleBox.cs

> A swatch that shows a colour *including its transparency*, by compositing it over a fixed checkerboard.

**Needs** — [`Resources.Designer.cs`](../Properties/Resources.Designer.cs.md)
**Used by** — [`ColorPicker.Designer.cs`](ColorPicker/ColorPicker.Designer.cs.md) · [`ColorPicker.cs`](ColorPicker/ColorPicker.cs.md) · [`Resources.Designer.cs`](../Properties/Resources.Designer.cs.md)
**Tier floor** — T3: it owns a paint callback and a compositing mode; everything above that is ordinary widget code.

## Purpose

Filling a rectangle with a half-transparent colour over the window's background tells the author nothing — they see the colour the host panel happens to be. This control fixes the backdrop: it tiles a checkerboard image it owns and alpha-composites the sample over it, so the same alpha always reads the same way regardless of where the swatch is placed.

## State

```text
RECORD ColorSampleBox
  color : Color        # the sample, alpha significant
```

**Invariants** — the backdrop is not settable. The control assigns its own tiled checkerboard at construction and refuses writes to the background image and its layout mode, because an author-chosen backdrop would defeat the entire purpose of the control.

## `ColorSample`

**Contract** — reads and writes the displayed colour. An identical write is dropped; a different one marks the surface for repaint rather than painting immediately, so a burst of slider movements costs one repaint.

## Painting

**Contract** — the sample is drawn over the already-painted checkerboard with source-over compositing, so alpha shows as the checkerboard showing through.

**Notes** — the control also asks its host for a composited (flicker-free) surface. That is the problem, not the mechanism: a swatch repainted continuously while a slider is dragged tears visibly if each repaint clears before drawing. A rebuild solves it with whatever double-buffering its toolkit offers.
