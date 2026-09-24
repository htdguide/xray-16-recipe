# src/editors/xrSdkControls/Controls/IntegerSlider/IntegerSlider.Designer.cs

> The pairing's layout: track bar on the left, spin box on the right, one row high and stretchable.

**Needs** — [`IntegerSlider.cs`](IntegerSlider.cs.md) · [`IntegerUpDown.cs`](../IntegerUpDown/IntegerUpDown.cs.md)
**Used by** — [`IntegerSlider.cs`](IntegerSlider.cs.md)
**Tier floor** — T4: a declarative layout description.

## Purpose

Generated layout for [`IntegerSlider`](IntegerSlider.cs.md). Three of its decisions are load-bearing.

## State

```text
RECORD IntegerSliderLayout
  track   : fills the left, stretches horizontally, no border
  spinner : fixed width, right-aligned, anchored to the right edge
  height  : one text row, fixed
  width   : stretches, with a floor and a ceiling
```

## Notes

**Load-bearing**: the control is exactly one text row tall and its height does not vary, because these appear as cell editors stacked in a grid and an uneven row height reads as a bug; the spinner is anchored to the right edge so widening the control gives the extra space to the track, which is the part that benefits; and both a minimum and a maximum width are declared, the maximum being a large but finite number rather than unbounded, so that a control docked into a very wide panel does not stretch a slider across an entire monitor.

**Not load-bearing**: every pixel coordinate, and the specific ceiling value, which is a round number chosen to be "wider than any panel" rather than measured.
