# src/editors/xrSdkControls/Controls/NumericSlider/NumericSlider.Designer.cs

> Layout for the fractional pairing: identical to the integer one, over a stock fractional spin box.

**Needs** — [`NumericSlider.cs`](NumericSlider.cs.md)
**Used by** — [`NumericSlider.cs`](NumericSlider.cs.md)
**Tier floor** — T4: a declarative layout description.

## Purpose

Generated layout for [`NumericSlider`](NumericSlider.cs.md), and byte-for-byte the same arrangement as [the integer slider's](../IntegerSlider/IntegerSlider.Designer.cs.md) — track bar filling the left, fixed-width spin box anchored right, one row tall, minimum and maximum widths declared.

## State

```text
RECORD NumericSliderLayout
  track   : fills the left, stretches, no border
  spinner : the host toolkit's stock fractional spin box, fixed width, right-anchored
  height  : one text row
```

## Notes

The only substantive difference from the integer variant is which spin box is embedded: this one uses the host toolkit's stock fractional control, where the integer variant embeds [the project's own](../IntegerUpDown/IntegerUpDown.cs.md). A rebuild with one spin box parameterised by value type collapses the two layouts into one.
