# src/editors/xrSdkControls/Controls/IntegerSlider/IntegerSlider.cs

> A track bar and a spin box editing one whole number, each correcting the other exactly once.

**Needs** — [`IntegerUpDown.cs`](../IntegerUpDown/IntegerUpDown.cs.md) · [`IntegerSlider.Designer.cs`](IntegerSlider.Designer.cs.md)
**Used by** — [`IntegerSlider.Designer.cs`](IntegerSlider.Designer.cs.md)
**Tier floor** — T3: two widgets and an agreement between them.

## Purpose

Authoring wants both gestures: drag for the feel of the range, type for the exact value. This pairs a track bar with a [spin box](../IntegerUpDown/IntegerUpDown.cs.md) over a single integer and makes them agree.

It is worth its own file because the agreement is not trivial: naively forwarding each child's change to the other recurses forever.

## State

```text
RECORD IntegerSlider
  spinner  : IntegerUpDown
  track    : TrackBar
  # no stored value: the spinner is authoritative and the track mirrors it
```

**Invariants** — `spinner.value == track.value` whenever no edit is in flight, and both children carry the same range. Range and value writes always go to both children. The control has no field of its own for the value, which is what makes divergence impossible to express.

## The echo break

**Contract** — each child's change notification checks whether the *other* child already holds that value, and returns if so. A write that originates in one child therefore travels exactly one hop.

```text
ON spinner_changed
  IF track.value == spinner.value THEN RETURN    # this is the echo of our own write
  track.value = spinner.value                    # which fires track_changed, which returns
  announce_value_changed()

ON track_changed
  IF spinner.value == track.value THEN RETURN
  spinner.value = track.value
  announce_value_changed()
```

**Notes** — comparing values rather than raising a suppression flag is the decision. It works here because both children hold the *same* integer with no conversion between them, so equality is exact. Its sibling [`NumericSlider`](../NumericSlider/NumericSlider.cs.md) cannot use it — a fractional value maps lossily onto an integer track position — and needs a flag instead.

## `SpinnerWidth`

**Contract** — moves the boundary between the spin box and the track bar, taking width from one and giving it to the other so the control's total width is unchanged. Refuses to shrink either child below its own minimum; a request that would is reduced to the largest legal one.

```text
FUNCTION set_spinner_width(requested)
  requested = max(requested, spinner.min_width)
  IF total_width - requested < track.min_width
    requested = total_width - track.min_width
  shift the spinner left (or right) by the delta and resize both
```

**Invariants** — the two children exactly fill the control, always. That is why the property is a single number rather than two.

## `MinimumSize`

**Contract** — the control's own minimum width is raised to at least the sum of its children's minima, whatever the caller asks for. A caller that asks for less gets the sum. This is why the constructor writes an empty size to this property: the write is discarded but the floor is computed, so the floor is correct before the control is ever laid out.

## Forwarded properties

**Contract** — range and current value go to both children; display base goes to the spin box; small step, large step, tick spacing and tick style go to the track bar. Each is a pure forward and exists so a caller configures one control rather than two.
