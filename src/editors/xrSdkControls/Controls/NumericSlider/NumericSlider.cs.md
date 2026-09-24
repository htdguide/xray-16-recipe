# src/editors/xrSdkControls/Controls/NumericSlider/NumericSlider.cs

> A track bar and a spin box editing one fractional number — which means the track's position is a lossy projection of the value, and the two cannot simply compare.

**Needs** — [`NumericSlider.Designer.cs`](NumericSlider.Designer.cs.md)
**Used by** — [`ColorPicker.Designer.cs`](../ColorPicker/ColorPicker.Designer.cs.md) · [`ColorPicker.cs`](../ColorPicker/ColorPicker.cs.md) · [`NumericSlider.Designer.cs`](NumericSlider.Designer.cs.md)
**Tier floor** — T3: a fixed-point projection and two widgets.

## Purpose

The same pairing as [`IntegerSlider`](../IntegerSlider/IntegerSlider.cs.md), for values that are not whole numbers — which is most of the weather model: densities, intensities, blend factors, distances. Almost every real-valued row in the weather grid is one of these.

The difference from the integer case is the whole content of the file. A track bar has integer positions. A fog density does not. So the track holds a *projection* of the value, and the control must decide the projection's resolution, must round when projecting, and must not treat a round-trip through the projection as a change.

## State

```text
RECORD NumericSlider
  spinner          : a fractional spin box        # authoritative
  track            : TrackBar                     # positions 0 .. slider_precision
  slider_precision : int                          # the track's upper position
  suppress_echo    : bool
```

**Invariants** — the spinner holds the value; the track holds `round((value - minimum) / (maximum - minimum) * slider_precision)`. The reverse map is exact in the other direction but not a round trip: many values project to one position. Range writes go to both, and re-project the current value.

## The projection

```text
FUNCTION slider_position_to_value() -> real
  RETURN minimum + (track.position / slider_precision) * (maximum - minimum)

FUNCTION set_slider_position_from(value : real)
  suppress_echo = true
  track.position = truncate((value - minimum) / (maximum - minimum) * slider_precision)
  suppress_echo = false
```

**Notes** — the forward map truncates rather than rounds, so the track sits at or just below the true value. It is not observable — the spinner shows the real value and the track is a visual — but a rebuild that reads the value back from the track would drift downwards.

`slider_precision` is the track's upper position, and it is a *property of the control*, not of the value. That is the decision worth keeping: a fixed pixel-to-value mapping would make a 0-to-1 property and a 0-to-1000 property feel different under the same drag. Setting it to a few hundred makes any range drag the same way.

**The division has no guard against a zero-width range.** A caller that sets minimum equal to maximum gets a division by zero here. Nothing in the editor does, but nothing prevents it either.

## The echo break

**Contract** — unlike the integer pairing, comparison alone is not enough: writing a value into the track and reading it back does not yield the value. So a suppression flag brackets every write the control makes into its own children, and each child's handler returns while it is set.

```text
ON spinner_changed
  IF suppress_echo OR slider_position_to_value() == spinner.value THEN RETURN
  set_slider_position_from(spinner.value)
  announce_value_changed()

ON track_changed
  IF suppress_echo OR spinner.value == slider_position_to_value() THEN RETURN
  suppress_echo = true
  spinner.value = slider_position_to_value()     # a drag snaps the value to the projection
  suppress_echo = false
  announce_value_changed()
```

**Notes** — dragging the track *quantizes* the value to the projection's resolution; typing into the spinner does not. That asymmetry is correct and intended: a drag is a coarse gesture and the author who needs the exact value types it.

## `SpinnerWidth`, `MinimumSize`

**Contract** — identical to the integer pairing: the boundary between the two children moves without changing the total width, refusing to shrink either below its own minimum; and the control's minimum width floors at the sum of its children's.

**Notes** — this logic is duplicated verbatim between the two sliders. The split between them is genuinely arbitrary at that level; a rebuild with one generic pairing over a value type writes it once.

## Forwarded properties

Range, value, display base, text alignment, decimal places to the spin box; tick spacing and tick style to the track.
