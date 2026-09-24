# src/editors/xrSdkControls/Controls/NumericSpinner

## What this module is responsible for

Editing a fractional number with an *unbounded* drag: grab a grip beside the number, move the mouse sideways, and the value follows without ever running out of screen, because the pointer is hidden and warped back to where it started.

## Where it sits and what it rests on

It rests on the host toolkit's fractional spin box for the numeric half, and on the windowing layer's ability to hide and reposition the pointer for the gesture.

## The load-bearing ideas

**Hide the pointer and warp it.** These two together are the gesture. Warping alone shows the cursor jumping backwards, which reads as a stutter; hiding alone still runs out of screen. A rebuild whose windowing layer offers a relative-motion capture mode gets both at once and should use it.

**Sensitivity is a property of the row, in value units per pixel.** Each property decides what a pixel of drag is worth in its own units — the same decision [`IIncrementable`](../Interfaces/IIncrementable.cs.md) encodes for the grid's own middle-drag.

## The twins

| File | Role |
|---|---|
| [`NumericSpinner.cs`](NumericSpinner.cs.md) | The infinite drag: pixel-to-value conversion, clamping, pointer hiding and recentring |
| [`NumericSpinner.Designer.cs`](NumericSpinner.Designer.cs.md) | The layout, and the three gesture bindings that live only here |

A resource file accompanies the layout; it holds no decision and has no twin.

## What the twins record as defects

The drag anchor is not re-read after the pointer is warped, so the first movement after each warp is measured from a stale origin; and the accumulation counter that decides when to warp is signed against a threshold that makes the sign meaningless. Both are documented in [`NumericSpinner.cs`](NumericSpinner.cs.md) and neither should be reproduced.
