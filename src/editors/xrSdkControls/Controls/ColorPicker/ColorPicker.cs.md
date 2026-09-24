# src/editors/xrSdkControls/Controls/ColorPicker/ColorPicker.cs

> Four channel sliders and a live sample swatch, agreeing on one colour without echoing each other into a loop.

**Needs** — [`ColorSampleBox.cs`](../ColorSampleBox.cs.md) · [`NumericSlider.cs`](../NumericSlider/NumericSlider.cs.md) · [`ColorPicker.Designer.cs`](ColorPicker.Designer.cs.md)
**Used by** — [`ColorPicker.Designer.cs`](ColorPicker.Designer.cs.md)
**Tier floor** — T3: widget composition and event wiring; nothing here touches a device or a byte layout.

## Purpose

The weather data model is mostly colours — ambient, hemisphere, fog, sun, thunderbolt. This is the widget that edits one. It is four independent 0–255 channel sliders plus a swatch, and its whole job is to keep those five views of one value consistent while the author drags any of them.

It is a separate control rather than part of the grid's colour cell because the same picker is also used standalone in editor panels.

## State

```text
RECORD ColorPicker
  channels        : list<Slider>    # red, green, blue, alpha, each 0..255 integer-valued
  sample          : ColorSampleBox  # the authoritative current value
  suppress_echo   : bool            # true while the control is writing to its own children
  alpha_enabled   : bool
  hexadecimal     : bool            # display base for all four channels at once
  text_alignment  : alignment       # forwarded to all four channels
```

**Invariants** — the swatch is the single source of truth: `Value` reads it and never a slider. While `suppress_echo` holds, child change notifications are dropped. When alpha is disabled, the alpha channel is forced to full and hidden, so a disabled alpha never silently multiplies the colour by a stale value.

## `Value`

**Contract** — reads and writes the whole colour. Writing pushes each channel into its slider with echo suppressed, sets the swatch, then recomputes once; an identical write is a no-op, which is what stops a grid that repaints on every change from ping-ponging with the control.

```text
FUNCTION set_value(new : Color)
  IF new == sample THEN RETURN          # idempotent write: breaks the repaint loop
  suppress_echo = true
  IF alpha_enabled THEN alpha.value = new.a
  red.value = new.r ; green.value = new.g ; blue.value = new.b
  sample = new
  suppress_echo = false
  recompute()                           # fires the change notification exactly once
```

## `recompute` (the reconciliation step)

**Contract** — assembles a colour from the four sliders, and if it differs from the swatch, stores it and announces the change. Called from every slider's change notification and once at first display.

**Invariants** — it returns immediately while `suppress_echo` holds, and it compares before storing. Those two guards together are the only thing preventing unbounded recursion between the sliders, the swatch and any listener that writes `Value` back.

## `AlphaEnabled`

**Contract** — hides or shows the alpha row, forces alpha to full when it is turned off, and slides the hexadecimal toggle up or down to close the gap the hidden row leaves.

**Notes** — the shift is a fixed pixel count rather than a relayout. A rebuild that lays this panel out with a row container gets the same effect by removing the row.

## `Hexadecimal`, `TextAlign`

**Contract** — pure broadcasts: set the property on all four channel sliders. They exist so that a caller configures the picker, not four sliders. Hexadecimal is also mirrored into the visible checkbox so the two agree when set programmatically.

## Notes

The hexadecimal text field that would let an author type `RRGGBBAA` directly is present in the layout but permanently hidden — the source carries an open note that it is unimplemented. A rebuild should either finish it or delete it; it is the one obvious gap in this control.
